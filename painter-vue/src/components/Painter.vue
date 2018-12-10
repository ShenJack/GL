<template>
    <div>
        <div @click="value1 = !value1" class="top">工具</div>
        <div>
            <div>
                <canvas id="glCanvas" :width="width" :height="height" @mousedown="mouseDown" @mousemove="mouseMove"
                        @mouseup="mouseUp"></canvas>
            </div>
            <Divider style="height: 100%" type="vertical"></Divider>

        </div>
        <colorSelect :showModal="showColor" @ok="colorOk" @cancel="colorCancel"></colorSelect>
        <Drawer :width="350" @keyup.space="value1 = !value1" title="工具" :closable="false" v-model="value1">
            <div style="margin-left: 20px;width: 280px">
                <Form label-position="right">
                    <Divider orientation="left">笔触</Divider>
                    <FormItem label="类型">
                        <RadioGroup @on-change="changePen" v-model="pen" type="button" size="large">
                            <Radio :label="STRING_PEN"><i class="icon-pen iconfont"/></Radio>
                            <Radio :label="STRING_PENCIL"><i class="icon-pencil iconfont"/></Radio>
                            <Radio :label="STRING_HIGHLIGHTER"><i class="icon-highlighter iconfont"/></Radio>
                        </RadioGroup>
                        <p>已选择 <span style="font-weight: bold">{{pen}}</span></p>
                    </FormItem>
                    <FormItem label="颜色">
                        <div style="border:black 1px solid;display: inline-block;margin-right: 5px"
                             :style="{background:paintConfig.color,width:'50px',height:'20px'}"></div>
                        <Button @click="showColorSelect(PAINT_COLOR)" :disabled="pen === STRING_PENCIL">选择颜色</Button>
                    </FormItem>
                    <FormItem label="粗细">
                        <div style="width: 200px;margin-left: 50px">
                            <Slider v-model="paintConfig.width" :min="1" :max="paintConfig.maxWidth"></Slider>
                            <Input v-model="paintConfig.width"/></div>
                    </FormItem>
                    <Divider orientation="left">画板</Divider>
                    <FormItem label="操作">
                        <Button @click="clearCanvas">清空</Button>
                    </FormItem>
                    <FormItem label="背景色">
                        <div style="border:black 1px solid;display: inline-block;margin-right: 5px"
                             :style="{background:boardConfig.color,width:'50px',height:'20px'}"></div>
                        <Button @click="showColorSelect(BACKGROUND_COLOR)">选择颜色</Button>
                    </FormItem>
                    <FormItem label="尺">
                        <Button @click="addRuler" :disabled="ruler.enabled">添加</Button>
                    </FormItem>
                </Form>
            </div>
        </Drawer>
        <div class="ruler" @mousedown="RulerMouseDown" @mousemove="RulerMouseMove" @mousewheel="RulerMouseWheel"
             :style="{top:ruler.y + 'px',left:ruler.x + 'px', transform: 'rotate(' + ruler.angle +  'deg)'}"
             @mouseup="RulerMouseUp">
        </div>
    </div>
</template>
<script>
    var paint = undefined;
    var canvas = undefined;
    var canvasLeft = 0;
    var canvasTop = 0;

    import colorSelect from './ColorSelect.vue'

    export default {
        name: 'painter',
        components: {
            colorSelect
        },
        data() {
            return {
                ruler: {
                    enabled: false,
                    x: 100,
                    y: 10,
                    angle: 0,
                },
                img: undefined,
                patternAvailable: false,
                value1: true,
                width: 1920,
                height: 800,

                PAINT_COLOR: "paint_color",
                BACKGROUND_COLOR: "board_color",

                showColor: false,
                paintConfig: {
                    color: "#4EB365",
                    show: "",
                    width: 10,
                    waitColor: false,
                    maxWidth: 100,
                },
                boardConfig: {
                    color: "#fff",
                    waitColor: false,
                },
                pen: "",
                STRING_PEN: '钢笔',
                STRING_PENCIL: '铅笔',
                STRING_HIGHLIGHTER: '荧光笔',

                startX: 0,
                startY: 0,
                down: false,

                rulerStartX: 0,
                rulerStartY: 0,

            }
        },
        methods: {
            changePen(pen) {
                if (pen === this.STRING_PENCIL) {
                    this.paintConfig.maxWidth = 20;
                    if (this.paintConfig.width > this.paintConfig.maxWidth) {
                        this.paintConfig.width = 20;
                    }
                } else if (pen === this.STRING_PEN) {
                    this.paintConfig.maxWidth = 100;
                }
            },
            initializeCanvas() {
                this.img = new Image();
                this.img.src = './paint.jpg';
                this.img.onload = function () {
                    this.patternAvailable = true;
                };
                canvas = document.getElementById("glCanvas");
                paint = canvas.getContext("2d");
                canvasLeft = canvas.getBoundingClientRect().left
                canvasTop = canvas.getBoundingClientRect().top
//                ctx.fillStyle="#FF0000";
//                ctx.fillRect(0,0,150,75);
            },

            pencilDown(event) {
                let paint = this.getPaint();
                let width = this.paintConfig.width;
                let x = event.clientX - canvasLeft;
                let y = event.clientY - canvasTop;
//                paint.lineWidth = 1;
//                for (let i = 0; i < Math.pow(width, 2) * Math.PI / 10; i++) {
//                    let angle = Math.random() * Math.PI * 2;
//                    let l = Math.random() * width;
//                    let y_local = y + l * Math.sin(angle);
//                    let x_local = x +  l * Math.cos(angle);
//                    paint.moveTo(x_local, y_local);
//                    paint.lineTo(x_local + 1, y_local + 1);
//                    paint.stroke();
//                }
//                paint.beginPath();
//                if(this.patternAvailable){
//                paint.fillStyle = paint.createPattern(this.img, 'no-repeat')
//                }
//                paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI)
//                paint.closePath();
//                paint.fill();
//                paint.drawImage(this.img, x - width / 2, y - width / 2, width, width)
                this.circleImg(paint, this.img, x, y, width);
            },
            mouseDown(event) {
                this.down = true;
                this.startX = event.clientX - canvasLeft;
                this.startY = event.clientY - canvasTop;
                if (this.pen === this.STRING_PEN) {
                    let paint = this.getPaint()
                    paint.beginPath();
                    paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI)
                    paint.closePath();
                    paint.fill();
                } else if (this.pen === this.STRING_PENCIL) {
                    this.pencilDown(event);
                } else {
                    let paint = this.getPaint();
                    let width = this.paintConfig.width;
                    paint.fillRect(this.startX, this.startY - width / 2, width / 4, width)
                }
            },
            mouseMove(event) {
                if (this.rulerDragging) {
                    let currentX = event.clientX - canvasLeft;
                    let currentY = event.clientY - canvasTop;
                    this.ruler.x += currentX - this.rulerStartX;
                    this.ruler.y += currentY - this.rulerStartY;

                    this.rulerStartX = currentX
                    this.rulerStartY = currentY
                    return
                }
                if (this.down) {
                    let currentX = event.clientX - canvasLeft;
                    let currentY = event.clientY - canvasTop;
                    let paint = this.getPaint();
                    let width = this.paintConfig.width;
                    switch (this.pen) {
                        case this.STRING_PEN:
                            paint.beginPath();
                            paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI);
                            paint.closePath();
                            paint.fill();

                            paint.moveTo(this.startX, this.startY);
                            paint.beginPath();

                            let rate = width / Math.sqrt(Math.pow(currentX - this.startX, 2) + Math.pow(currentY - this.startY, 2))
                            if ((currentX - this.startX) * (currentY - this.startY) > 0) {
                                //同号 向右下或左上
                                paint.lineTo(this.startX - Math.abs(currentY - this.startY) * rate, this.startY + Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(currentX - Math.abs(currentY - this.startY) * rate, currentY + Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(currentX + Math.abs(currentY - this.startY) * rate, currentY - Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(this.startX + Math.abs(currentY - this.startY) * rate, this.startY - Math.abs(currentX - this.startX) * rate);
                            } else {
                                paint.lineTo(this.startX + Math.abs(currentY - this.startY) * rate, this.startY + Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(currentX + Math.abs(currentY - this.startY) * rate, currentY + Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(currentX - Math.abs(currentY - this.startY) * rate, currentY - Math.abs(currentX - this.startX) * rate);
                                paint.lineTo(this.startX - Math.abs(currentY - this.startY) * rate, this.startY - Math.abs(currentX - this.startX) * rate);
                            }
                            let s = [];

                            paint.lineTo(this.startX, this.startY);
                            paint.closePath();
                            paint.fill();

                            console.log(s.join(' '))


                            paint.moveTo(currentX, currentY);
                            paint.beginPath();
                            paint.arc(currentX, currentY, paint.lineWidth, 0, 2 * Math.PI);
                            paint.closePath();
                            paint.fill();
                            break;
                        case this.STRING_PENCIL:
                            this.circleImg(paint, this.img, currentX, currentY, width);
                            break;
                        case this.STRING_HIGHLIGHTER:
                            paint.lineWidth = width;
                            paint.fillRect(this.startX, this.startY - width / 2, width / 4, width);
                            paint.moveTo(this.startX, this.startY);
                            if (Math.abs(currentX - this.startX) > width / 4) {
                                paint.lineTo(currentX, currentY);
                                paint.stroke();
                            }
                            paint.fillRect(currentX, currentY - width / 2, width / 4, width);

                    }

                    this.startX = currentX;
                    this.startY = currentY;

                    if (currentX >= this.width || currentY >= this.height) {
                        this.down = false
                    }
                }
            },
            mouseUp(event) {
                this.down = false
            },
            clearCanvas() {
                let paint = this.getPaint();
                paint.fillStyle = this.boardConfig.color;
                paint.fillRect(0, 0, this.width, this.height);
            },
            getPaint() {
                paint.lineWidth = this.paintConfig.width;
                paint.strokeStyle = this.paintConfig.color;
                paint.fillStyle = this.paintConfig.color;
                return paint;
            },
            circleImg(paint, img, currentX, currentY, width) {
                paint.save();
                paint.beginPath();
                paint.arc(currentX, currentY, width / 2, 0, 2 * Math.PI);
                paint.closePath();
                paint.clip();
                paint.drawImage(img, Math.random() * (img.width - width), Math.random() * (img.height - width), width, width, currentX - width / 2, currentY - width / 2, width, width);
//                paint.drawImage(img,currentX-width/2, currentY-width/2, width, width);
                paint.restore();
            },


            colorOk(color) {
                this.showColor = false;
                if (this.paintConfig.waitColor) {
                    this.paintConfig.color = color;
                    this.paintConfig.waitColor = false;
                } else if (this.boardConfig.waitColor) {
                    this.boardConfig.color = color;
                    this.boardConfig.waitColor = false;
                    this.clearCanvas()

                }
            },
            colorCancel() {
                this.showColor = false;
            },
            showColorSelect(color) {
                this.showColor = true;
                switch (color) {
                    case this.PAINT_COLOR:
                        this.paintConfig.waitColor = true;
                        break;
                    case this.BACKGROUND_COLOR:
                        this.boardConfig.waitColor = true;
                }
            },
            addRuler() {
                this.ruler.enabled = true
            },
            RulerMouseDown(event) {
                this.rulerStartX = event.clientX - canvasLeft;
                this.rulerStartY = event.clientY - canvasTop;
                this.rulerDragging = true;

            },
            RulerMouseMove(event) {
                // 当画笔到了尺子上
                if(this.down){
                    this.down = false
                }
                if (this.rulerDragging) {
                    let currentX = event.clientX - canvasLeft;
                    let currentY = event.clientY - canvasTop;
                    this.ruler.x += currentX - this.rulerStartX;
                    this.ruler.y += currentY - this.rulerStartY;

                    this.rulerStartX = currentX
                    this.rulerStartY = currentY

                }
            },
            RulerMouseUp(event) {
                this.rulerDragging = false;
            },
            RulerMouseWheel(event){
                this.ruler.angle += event.wheelDelta/120;
            },
        },
        mounted() {
            this.initializeCanvas()
            this.pen = this.STRING_HIGHLIGHTER;
            document.onkeydown = function (e) {
                let key = window.event.keyCode;
                if (key === 32) {
                    this.value1 = !this.value1;
                }
            };
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    canvas {
    }

    .top {
        cursor: pointer;
        padding: 10px;
        background: rgba(0, 153, 229, .7);
        color: #fff;
        text-align: center;
        border-radius: 2px;
        display: block;
        position: fixed;
        right: 0;
        top: 0;
    }

    .ruler {
        cursor: move;
        position: fixed;
        z-index: 1000;
        width: 120px;
        height: 700px;
        text-align: center;
        background: rgba(75, 75, 75, 0.47);
        border: #7a7a7a 4px solid;
    }
</style>
