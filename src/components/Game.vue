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
          height: 128,
          width: 256
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
      }
  },
  props: ['map'],
  name: 'Game',
  methods: {
    rotate(angle) {
      this.player.direction = (this.player.direction + angle + (2 * Math.PI)) % (2 * Math.PI)
    },
    walk(distance) {
      var dx = Math.cos(this.player.direction) * distance
      var dy = Math.sin(this.player.direction) * distance
      if (this.get(this.player.x + dx, this.player.y) <= 0) this.player.x += dx
      if (this.get(this.player.x, this.player.y + dy) <= 0) this.player.y += dy
    },
    get(x, y) {     
      x = Math.floor(x)
      y = Math.floor(y)
      if (x < 0 || x > this.map.size - 1 || y < 0 || y > this.map.size - 1) return -1
      return this.map.wallGrid[y * this.map.size + x]
    },
    cast(angle) {
      var sin = Math.sin(angle)
      var cos = Math.cos(angle)

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
      var dx = cos < 0 ? shiftX : 0
      var dy = sin < 0 ? shiftY : 0
      var t = this.get(step.x - dx, step.y - dy)
      step.edge = false
      step.hit = t
      step.distance = distance + Math.sqrt(step.length2)
      step.offset = offset - Math.floor(offset)
      return step
    },
    loop(time) {      
      if (this.canvas.context) {        
        this.canvas.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
        var spacing = this.canvas.width / this.resolution
        for (var column = 0; column < this.resolution; column++) {
          var x = column / this.resolution - 0.5
          
          var angle = Math.atan2(x, this.focalLength)
          var ray = this.cast(this.player.direction + angle)
          if (ray.hit === 1) {
            var z_inv = 1 / (ray.distance * Math.cos(angle))
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
        this.rotate(-Math.PI * seconds)
      } else if (this.controls.right) {
        this.rotate(Math.PI * seconds)
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
