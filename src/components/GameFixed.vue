<template>
  <div>
    <canvas ref="game" style="border:1px solid #000000;" />
    <img id="wall" src="../assets/wall.png" style="display: none;">
  </div>
</template>

<script>
export default {
  data() {
    return {
        wallTexture: null,
        canvas: {
          context: null,
          height: 256,
          width: 512
        },       
        player: {
          x: 10,
          y: 10,
          direction: 0
        },
        controls : {
          left: false,
          forward: false,
          right: false,
          backwards: false
        },
        id: undefined,
        focalLength: 0.8,
        resolution: 128,        
        lastTime: 0,
        trigQuarters: null,
        atan2FocalLength: null   
      }
  },
  props: ['map'],
  name: 'GameFixed',
  methods: {
    buildLookupTables() {
        /* // FROM https://gitlab.com/tllado/fastTrig/-/blob/master/fastTrig.c
        let trigQuarters = new Uint8Array([128, 131, 134, 137, 140, 144, 147, 150, 153, 156, 159, 162, 165, 168, 171, 
            174, 177, 179, 182, 185, 188, 191, 193, 196, 199, 201, 204, 206, 209, 211, 
            213, 216, 218, 220, 222, 224, 226, 228, 230, 232, 234, 235, 237, 239, 240, 
            241, 243, 244, 245, 246, 248, 249, 250, 250, 251, 252, 253, 253, 254, 254, 
            254, 255, 255, 255, 255])

        this.trigQuarters = trigQuarters
        */

        let trigQuarters = new Uint8Array(65)
        for (var i = 0; i < 65; i++) {
            let x = Math.round(Math.sin((i / 256.0) * Math.PI * 2.0) * 127.0) + 128
            trigQuarters[i] = x
        }

        this.trigQuarters = trigQuarters


        let atan2FocalLength = new Uint8Array(128)
        for (i = 0; i < 128; i++) {
           var x = Math.round((Math.atan2(i / 128 - 0.5, 0.8) / (2 * Math.PI))*256.0) + 128           
           atan2FocalLength[i] = x
        }
        this.atan2FocalLength = atan2FocalLength

        /*
        for (var i = -(Math.PI * 2); i < (Math.PI * 2); i += 0.1) {
            console.log(i)
            console.log(Math.sin(i))
            console.log(this.sin(i))
            console.log('')
        }
        */
    },
    sin(a) {
        // For now rounding because the 0..256 angle is decimal so the rotations match
        let x = Math.round(a) % 256

        var v = 0
        if (x < 64)
            v = this.trigQuarters[x]
        else if (x < 128)
            v = this.trigQuarters[128 - x]
        else if (x < 192)
            v = 256 - this.trigQuarters[x - 128]
        else 
            v = 256 - this.trigQuarters[256 - x]
                    
        // Convert (THIS NEEDS TO BE REWRITTEN FOR FIXED POINT)
        return (v - 128) / 127.0
    },
    cos(a) {
        // For now rounding because the 0..256 angle is decimal so the rotations match
        let x = Math.round(a) % 256

        var v = 0
        if (x < 64)
            v = this.trigQuarters[64 - x]
        else if (x < 128)
            v = 256 - this.trigQuarters[x - 64]
        else if (x < 192)
            v = 256 - this.trigQuarters[192 - x]
        else 
            v = this.trigQuarters[x - 192]
                    
        // Convert (THIS NEEDS TO BE REWRITTEN FOR FIXED POINT)
        return (v - 128) / 127.0
    },
    atan2FL(x) {
        let v = this.atan2FocalLength[x]
        let a = v + 128
        return a % 256 
    },
    rotate(angle) {
      this.player.direction = (this.player.direction + angle + 256) % 256
      //console.log(`Player Direction: ${Math.round(this.player.direction)}`)
    },
    walk(distance) {
      // DISTANCE AND PLAYER.X and PLAYER.Y need to be fixed point (16-bit)
      var dx = this.cos(this.player.direction) * distance
      var dy = this.sin(this.player.direction) * distance
      if (this.get(this.player.x + dx, this.player.y) <= 0) this.player.x += dx
      if (this.get(this.player.x, this.player.y + dy) <= 0) this.player.y += dy
      //console.log(`Player X: ${this.player.x}`)
      //console.log(`Player Y: ${this.player.y}`)
    },
    get(x, y) {
      // x, and y need to be Fixed Point and floor needs to be re-written 
      x = Math.floor(x)
      y = Math.floor(y)
      if (x < 0 || x > this.map.size - 1 || y < 0 || y > this.map.size - 1) return -1
      return this.map.wallGrid[y * this.map.size + x]
    },
    cast(angle) {
      // This is actually ok?
      var sin = this.sin(angle)
      var cos = this.cos(angle)

      var step = {
        x: this.player.x,
        y: this.player.y,
        distance: 0,
        length2: 0,
        hit: 0
      }
      while (step.hit === 0) {
        var stepX = this.step(sin, cos, step.x, step.y)
        var stepY = this.step(cos, sin, step.y, step.x, true)
        step = stepX.length2 < stepY.length2
          ? this.inspect(sin, cos, stepX, 1, 0, step.distance, stepX.y) :
          this.inspect(sin, cos, stepY, 0, 1, step.distance, stepY.x)
      }

      return step
    },
    step(rise, run, x, y, inverted) {
      // Needs to be rewritten for Fixed Point
      if (run === 0) return { length2: Infinity }
      var dx = run > 0 ? Math.floor(x + 1) - x : Math.ceil(x - 1) - x
      var dy = dx * (rise / run)
      return {
        x: inverted ? y + dy : x + dx,
        y: inverted ? x + dx : y + dy,
        length2: dx * dx + dy * dy
      }
    },
    inspect(sin, cos, step, shiftX, shiftY, distance, offset) {
      // Needs to be rewritten for Fixed Point
      var dx = cos < 0 ? shiftX : 0 // These comparisons for negative cos/sin need to be updated
      var dy = sin < 0 ? shiftY : 0
      var t = this.get(step.x - dx, step.y - dy)
      step.edge = false
      step.hit = t
      step.distance = distance + Math.sqrt(step.length2)  // Need sqrt FixedPoint
      step.offset = offset - Math.floor(offset)
      return step
    },
    loop(time) {      
      if (this.canvas.context) {        
        this.canvas.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
        var spacing = this.canvas.width / this.resolution
        for (var column = 0; column < this.resolution; column++) {
          //var x = column / this.resolution - 0.5
          //var angle = Math.atan2(x, this.focalLength)
          var angle = this.atan2FL(column)
          var ray = this.cast(this.player.direction + angle)
          if (ray.hit === 1) {
            // This needs to be rewritten for Fixed Point
            var z_inv = 1 / (ray.distance * this.cos(angle))
            var wallHeight = this.canvas.height * z_inv
            var bottom = this.canvas.height / 2 * (1 + z_inv)
            var top = bottom - wallHeight
            var height = wallHeight

            var textureX = Math.floor(ray.offset * this.wallTexture.width)
            this.canvas.context.imageSmoothingEnabled = false
            this.canvas.context.drawImage(this.wallTexture, textureX, 0, 1, this.wallTexture.height, Math.floor(column * spacing), top, Math.ceil(spacing), height)
          }
        }
      }

      var seconds = (time - this.lastTime) / 1000;
      this.lastTime = time
      if (this.controls.left) {        
        this.rotate(-128.0 * seconds) // -128 is -Math.PI
      } else if (this.controls.right) {                
        this.rotate(128.0 * seconds) // 128 is Math.PI
      }
      if (this.controls.forward) {
        this.walk(3 * seconds)        
      } else if (this.controls.backwards) {
        this.walk(-3 * seconds)
      }

      this.id = requestAnimationFrame(this.loop)
    },
    onKeyDown(e) {
      switch (e.which) {
        case 37:
          this.controls.left = true
          break
        case 38:
          this.controls.forward = true
          break
        case 39:
          this.controls.right = true
          break
        case 40:
          this.controls.backwards = true
          break
      }
    },
    onKeyUp(e) {
      switch (e.which) {
        case 37:
          this.controls.left = false
          break
        case 38:
          this.controls.forward = false
          break          
        case 39:
          this.controls.right = false
          break
        case 40:
          this.controls.backwards = false
          break
      }
    },
  },
  mounted() {    
    var c = this.$refs.game
    this.canvas.context = c.getContext('2d')        
    c.width = this.canvas.width
    c.height = this.canvas.height
    this.wallTexture = document.getElementById('wall')
    this.buildLookupTables()
  },
  created() {
    this.id = requestAnimationFrame(this.loop)
    window.addEventListener('keydown', this.onKeyDown);
    window.addEventListener('keyup', this.onKeyUp);
  },
  destroyed() {
    cancelAnimationFrame(this.id)
    this.id = undefined
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#game {
  image-rendering: pixelated;
}
</style>
