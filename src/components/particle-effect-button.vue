<template>
  <div class="particles">
    <div ref="_wrapper"
      class="wrapper"
      :style="wrapperStyles">
      <div class="content"
        :style="contentStyles">
        <slot></slot>
      </div>
    </div>
    <canvas ref="_canvas"
      class="canvas"
      :style="canvasStyles"></canvas>
  </div>
</template>

<script>
import anime from 'animejs'
import raf from 'raf'
export default {
  name: 'ParticleEffectButton',
  props: {
    duration: {
      type: Number,
      default: 1000
    },
    direction: {
      type: String,
      default: 'left'
    },
    easing: {
      type: [String, Array],
      default: 'easeInOutCubic'
    },
    canvasPadding: {
      type: Number,
      default: 150
    },
    particlesAmountCoefficient: {
      type: Number,
      default: 3
    },
    hidden: {
      type: Boolean,
      default: false
    },
    oscillationCoefficient: {
      type: Number,
      default: 20
    },
    color: {
      type: String,
      default: '#000'
    },
    type: {
      type: String,
      default: 'circle'
    },
    drawStyle: {
      type: String,
      default: 'fill'
    },
    size: {
      type: Number,
      default: () => Math.floor(Math.random() * 3 + 1)
    },
    speed: {
      type: Number,
      default: () => rand(4)
    }
  },
  data () {
    return {
      status: this.hidden ? 'hidden' : 'normal',
      progress: 0,
      _progress: null,
      _particles: null,
      _rect: {
        width: 0,
        height: 0
      },
      _ctx: null,
      _raf: null,
      // dom
      _wrapper: null,
      _canvas: null,
      wrapperStyles: {},
      contentStyles: {},
      canvasStyles: {}
    }
  },
  watch: {
    hidden (newValue, oldValue) {
      const status = this.status
      if (status === 'normal' && newValue) {
        this.status = 'hiding'
        this._startAnimation()
      } else if (status === 'hidden' && !newValue) {
        this.status = 'showing'
        this._startAnimation()
      }
    }
  },
  methods: {
    _startAnimation () {
      const { duration, easing, canvasPadding } = this
      const { status } = this
      if (status === 'hiding') {
        this._progress = 0
      } else {
        this._progress = 1
      }
      this._particles = []
      this._rect = this._wrapper.getBoundingClientRect()
      this._canvas.width = this._rect.width + canvasPadding * 2
      this._canvas.height = this._rect.height + canvasPadding * 2
      this._ctx = this._canvas.getContext('2d')
      anime({
        targets: { value: status === 'hiding' ? 0 : 100 },
        value: status === 'hiding' ? 100 : 0,
        duration: duration,
        easing: easing,
        begin: this.onBegin,
        update: anim => {
          const value = anim.animatables[0].target.value
          setTimeout(() => {
            this.progress = value
          })
          if (duration) {
            this._addParticles(value / 100)
          }
        }
      })
    },
    _addParticles (progress) {
      const { canvasPadding, direction, particlesAmountCoefficient } = this
      const { status } = this
      const { width, height } = this._rect
      const delta =
        status === 'hiding'
          ? progress - this._progress
          : this._progress - progress
      const isHorizontal = this._isHorizontal()
      const progressValue =
        (isHorizontal ? width : height) * progress +
        delta * (status === 'hiding' ? 100 : 220)
      this._progress = progress
      let x = canvasPadding
      let y = canvasPadding
      if (isHorizontal) {
        x += direction === 'left' ? progressValue : width - progressValue
      } else {
        y += direction === 'top' ? progressValue : height - progressValue
      }
      let i = Math.floor(particlesAmountCoefficient * (delta * 100 + 1))
      if (i > 0) {
        while (i--) {
          this._addParticle({
            x: x + (isHorizontal ? 0 : width * Math.random()),
            y: y + (isHorizontal ? height * Math.random() : 0)
          })
        }
      }
      if (!this._raf) {
        this._raf = raf(this._loop)
      }
    },
    _addParticle (opts) {
      const { duration, size, speed } = this
      const { status } = this
      const frames = duration * 60 / 1000
      const _speed = isFunc(speed) ? speed() : speed
      const _size = isFunc(size) ? size() : size
      this._particles.push({
        startX: opts.x,
        startY: opts.y,
        x: status === 'hiding' ? 0 : _speed * -frames,
        y: 0,
        angle: rand(360),
        counter: status === 'hiding' ? 0 : frames,
        increase: Math.PI * 2 / 100,
        life: 0,
        death: status === 'hiding' ? frames - 20 + Math.random() * 40 : frames,
        speed: _speed,
        size: _size
      })
    },
    _loop () {
      this._updateParticles()
      this._renderParticles()
      if (this._particles.length) {
        this._raf = raf(this._loop)
      } else {
        this._raf = null
        this._cycleStatus()
        this.onComplete()
      }
      this.change()
    },
    _updateParticles () {
      const { oscillationCoefficient } = this
      const { status } = this
      for (let i = 0; i < this._particles.length; i++) {
        const p = this._particles[i]
        if (p.life > p.death) {
          this._particles.splice(i, 1)
        } else {
          p.x += p.speed
          p.y = oscillationCoefficient * Math.sin(p.counter * p.increase)
          p.life++
          p.counter += status === 'hiding' ? 1 : -1
        }
      }
    },
    _renderParticles () {
      const { color, type, drawStyle } = this
      const { status } = this
      this._ctx.clearRect(0, 0, this._canvas.width, this._canvas.height)
      this._ctx.fillStyle = this._ctx.strokeStyle = color
      for (let i = 0; i < this._particles.length; ++i) {
        const p = this._particles[i]
        if (p.life < p.death) {
          this._ctx.translate(p.startX, p.startY)
          this._ctx.rotate(p.angle * Math.PI / 180)
          this._ctx.globalAlpha =
            status === 'hiding' ? 1 - p.life / p.death : p.life / p.death
          this._ctx.beginPath()
          if (type === 'circle') {
            this._ctx.arc(p.x, p.y, p.size, 0, 2 * Math.PI)
          } else if (type === 'triangle') {
            this._ctx.moveTo(p.x, p.y)
            this._ctx.lineTo(p.x + p.size, p.y + p.size)
            this._ctx.lineTo(p.x + p.size, p.y - p.size)
          } else if (type === 'rectangle') {
            this._ctx.rect(p.x, p.y, p.size, p.size)
          }
          if (drawStyle === 'fill') {
            this._ctx.fill()
          } else if (drawStyle === 'stroke') {
            this._ctx.closePath()
            this._ctx.stroke()
          }
          this._ctx.globalAlpha = 1
          this._ctx.rotate(-p.angle * Math.PI / 180)
          this._ctx.translate(-p.startX, -p.startY)
        }
      }
    },
    _cycleStatus () {
      const { status } = this
      if (status === 'normal') {
        this.status = 'hiding'
      } else if (status === 'hidden') {
        this.status = 'showing'
      } else if (status === 'hiding') {
        this.status = 'hidden'
      } else if (status === 'showing') {
        this.status = 'normal'
      }
    },
    _isHorizontal () {
      return this.direction === 'left' || this.direction === 'right'
    },
    change () {
      const { direction } = this
      const { status, progress } = this
      this.wrapperStyles = {}
      this.contentStyles = {}
      this.canvasStyles = {}
      if (status === 'hiding' || status === 'showing') {
        const prop = this._isHorizontal() ? 'translateX' : 'translateY'
        const size = this._isHorizontal() ? this._rect.width : this._rect.height
        const value =
          direction === 'left' || direction === 'top' ? progress : -progress
        const px = Math.ceil(size * value / 100)
        this.$set(this.wrapperStyles, 'transform', `${prop}(${px}px)`)
        this.$set(this.contentStyles, 'transform', `${prop}(${-px}px)`)
      } else if (status === 'hidden') {
        this.$set(this.wrapperStyles, 'visibility', 'hidden')
        this.$set(this.canvasStyles, 'visibility', 'hidden')
      } else if (status === 'normal') {
        this.$set(this.canvasStyles, 'visibility', 'hidden')
      }
    },
    onBegin () {
      this.$emit('begin')
    },
    onComplete () {
      this.$emit('complete')
    }
  },
  mounted () {
    this._wrapper = this.$refs._wrapper
    this._canvas = this.$refs._canvas
  }
}
const rand = value => {
  return Math.random() * value - value / 2
}
const isFunc = value => {
  return typeof value === 'function'
}
</script>

<style scoped>
.particles {
  position: relative;
  display: inline-block;
}
.wrapper {
  position: relative;
  display: inline-block;
  overflow: hidden;
}
.content:focus,
.content > *:focus {
  outline: none;
}
.canvas {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate3d(-50%, -50%, 0);
  pointer-events: none;
}
</style>
