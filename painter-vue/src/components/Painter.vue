<template>
    <div style="height: 920px">
        <div style="display: flex;height: 100%">
            <div style="width: 1550px">
                <canvas id="glCanvas" :width="width" :height="height" @mousedown="mouseDown" @mousemove="mouseMove"
                        @drop="handleFileSelect"
                        @mouseup="mouseUp" @dragover="handleDragOver"></canvas>
                <canvas id="cover" :width="width" :height="height"></canvas>
            </div>
            <Divider style="height: 100%" type="vertical"></Divider>
            <div style="width: 350px">
                <div style="margin-left: 20px;width: 280px">
                    <Form label-position="right">
                        <Divider orientation="left">笔触</Divider>
                        <FormItem label="类型">
                            <RadioGroup @on-change="changePen" v-model="pen" type="button" size="large">
                                <Radio :label="STRING_PEN"><i class="icon-pen iconfont"/></Radio>
                                <Radio :label="STRING_PENCIL"><i class="icon-pencil iconfont"/></Radio>
                                <Radio :label="STRING_HIGHLIGHTER"><i class="icon-highlighter iconfont"/></Radio>
                                <Radio :label="STRING_ERASER"><i class="icon-xiangpi iconfont"/></Radio>
                            </RadioGroup>
                            <p>已选择 <span style="font-weight: bold">{{pen}}</span></p>
                        </FormItem>
                        <FormItem label="颜色">
                            <div style="border:black 1px solid;display: inline-block;margin-right: 5px"
                                 :style="{background:paintConfig.color,width:'50px',height:'20px'}"></div>
                            <Button @click="showColorSelect(PAINT_COLOR)" :disabled="pen === STRING_PENCIL">选择颜色
                            </Button>
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
                        <FormItem label="直尺">
                            <Button @click="addRuler" :disabled="ruler.enabled">添加</Button>
                            <Button @click="clearRuler" :disabled="!ruler.enabled">移除</Button>
                        </FormItem>
                        <FormItem label="图形">
                            <RadioGroup @on-change="onShape" v-model="shape" type="button" size="large">
                                <Radio :label="STRING_LINE">直线</Radio>
                                <Radio :label="STRING_CIRCLE">圆</Radio>
                                <Radio :label="STRING_SQUARE">方形</Radio>
                                <Radio :label="STRING_ZHENG">正方形</Radio>
                            </RadioGroup>
                        </FormItem>
                        <FormItem label="控制">
                            <Button @click="cancel">撤销</Button>
                        </FormItem>
                        <FormItem label="">
                            <Button @click="download">保存</Button>
                            <Button><a :href="link" download="picture.png">下载</a></Button>
                        </FormItem>
                    </Form>
                </div>
            </div>
        </div>
        <colorSelect :showModal="showColor" @ok="colorOk" @cancel="colorCancel"></colorSelect>

        <div v-if="ruler.enabled" class="ruler" @mousedown="RulerMouseDown" @mousemove="RulerMouseMove"
             @mousewheel="RulerMouseWheel"
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
    var reader;

    import colorSelect from './ColorSelect.vue'

    export default {
        name: 'painter',
        components: {
            colorSelect
        },
        data() {
            return {
                shape:"直线",
                STRING_LINE:"直线",
                STRING_CIRCLE:"圆",
                STRING_SQUARE:"方形",
                STRING_ZHENG:"正方形",
                canvas: undefined,
                ruler: {
                    enabled: false,
                    x: 100,
                    y: 10,
                    angle: 0,
                },
                img: undefined,
                patternAvailable: false,
                value1: true,
                width: 1550,
                height: 920,

                PAINT_COLOR: "paint_color",
                BACKGROUND_COLOR: "board_color",

                link: '',

                showColor: false,
                paintConfig: {
                    color: "#4EB365",
                    show: "",
                    width: 10,
                    waitColor: false,
                    maxWidth: 30,
                },
                boardConfig: {
                    color: "#fff",
                    waitColor: false,
                },
                pen: "",
                STRING_PEN: '钢笔',
                STRING_PENCIL: '铅笔',
                STRING_HIGHLIGHTER: '荧光笔',
                STRING_ERASER: '橡皮',

                startX: 0,
                startY: 0,
                down: false,

                circle: {
                    enable: false,
                    startX: 0,
                    startY: 0,
                },

                rulerStartX: 0,
                rulerStartY: 0,

                cancelList: [],
                cancelIndex: 0,

            }
        },
        computed: {},
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
                if (this.circle.enable) {
                    this.circle.startX = this.startX
                    this.circle.startY = this.startY
                    return
                }
                if (this.pen === this.STRING_PEN) {
                    let paint = this.getPaint()
                    paint.beginPath();
                    paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI)
                    paint.closePath();
                    paint.fill();
                } else if (this.pen === this.STRING_PENCIL) {
                    this.pencilDown(event);
                } else if (this.pen === this.STRING_HIGHLIGHTER) {
                    let paint = this.getPaint();
                    let width = this.paintConfig.width;
                    paint.fillRect(this.startX, this.startY - width / 2, width / 4, width)
                } else if (this.pen === this.STRING_ERASER) {
                    let paint = this.getPaint()
                    paint.fillStyle = this.boardConfig.color;
                    paint.beginPath();
                    paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI)
                    paint.closePath();
                    paint.fill();
                }
            },
            mouseMove(event) {
                if (this.circle.enable) {
                    this.circle.startX = this.startX
                    this.circle.startY = this.startY
                    return
                }
                let rate;
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

                            rate = width / Math.sqrt(Math.pow(currentX - this.startX, 2) + Math.pow(currentY - this.startY, 2));
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
                            paint.moveTo(currentX, currentY);
                            paint.beginPath();
                            paint.arc(currentX, currentY, paint.lineWidth, 0, 2 * Math.PI);
                            paint.closePath();
                            paint.fill();
                            break;
                        case this.STRING_ERASER:
                            paint.fillStyle = this.boardConfig.color;
                            paint.beginPath();
                            paint.arc(this.startX, this.startY, paint.lineWidth, 0, 2 * Math.PI);
                            paint.closePath();
                            paint.fill();

                            paint.moveTo(this.startX, this.startY);
                            paint.beginPath();

                            rate = width / Math.sqrt(Math.pow(currentX - this.startX, 2) + Math.pow(currentY - this.startY, 2));
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

                            paint.lineTo(this.startX, this.startY);
                            paint.closePath();
                            paint.fill();
                            paint.moveTo(currentX, currentY);
                            paint.beginPath();
                            paint.arc(currentX, currentY, paint.lineWidth, 0, 2 * Math.PI);
                            paint.closePath();
                            paint.fill();
                            break;
                        case this.STRING_PENCIL:
                            let delta = 0.8
                            rate = width / Math.sqrt(Math.pow(currentX - this.startX, 2) + Math.pow(currentY - this.startY, 2))
                            let steps = Math.abs(currentX - this.startX) > Math.abs(currentY - this.startY) ? Math.abs(currentX - this.startX) / delta :
                                Math.abs(currentY - this.startY) / delta
                            let deltaX = (currentX - this.startX) / steps;
                            let deltaY = (currentY - this.startY) / steps;
                            let x = this.startX;
                            let y = this.startY;
                            for (let i = 0; i < steps; i++) {
                                x += deltaX
                                y += deltaY
                                this.circleImg(paint, this.img, x, y, width);
                            }

                            break;
                        case this.STRING_HIGHLIGHTER:
                            delta = 0.8
                            steps = Math.abs(currentX - this.startX) > Math.abs(currentY - this.startY) ? Math.abs(currentX - this.startX) / delta :
                                Math.abs(currentY - this.startY) / delta;
                            deltaX = (currentX - this.startX) / steps;
                            deltaY = (currentY - this.startY) / steps;
                            x = this.startX;
                            y = this.startY;
                            for (let i = 0; i < steps; i++) {
                                x += deltaX
                                y += deltaY
                                paint.fillRect(x, y - width / 2, width / 4, width);
                            }
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
                let currentX = event.clientX - canvasLeft;
                let currentY = event.clientY - canvasTop;
                if (this.circle.enable) {
                    let paint = this.getPaint()
                    paint.beginPath();
                    paint.arc(this.circle.startX + (currentX - this.circle.startX) / 2, this.circle.startY + (currentY - this.circle.startY) / 2, Math.sqrt(Math.pow(currentX - this.startX, 2) + Math.pow(currentY - this.startY, 2)) / 2, 0, 2 * Math.PI);
                    paint.closePath();
                    paint.fill();
                }
                this.saveImageToAry();
            },
            clearCanvas() {
                let paint = this.getPaint();
                paint.fillStyle = this.boardConfig.color;
                paint.fillRect(0, 0, this.width, this.height);
                paint.fillStyle = this.paintConfig.color;
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

            rectImg(paint, img, x1, x2, y1, y2) {

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
            clearRuler() {
                this.ruler.enabled = false
            },
            RulerMouseDown(event) {
                this.rulerStartX = event.clientX - canvasLeft;
                this.rulerStartY = event.clientY - canvasTop;
                this.rulerDragging = true;

            },
            RulerMouseMove(event) {
                // 当画笔到了尺子上
                if (this.down) {
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
            RulerMouseWheel(event) {
                this.ruler.angle += event.wheelDelta / 120;
            },
            download() {
                this.link = document.getElementById('glCanvas').toDataURL()
            },
            cancel: function () {
                this.cancelIndex++;
                this.clearCanvas();
                var image = new Image();
                var index = this.cancelList.length - 1 - this.cancelIndex;
                var url = this.cancelList[index];
                image.src = url;
                image.onload = function () {
                    paint.drawImage(image, 0, 0, image.width, image.height, 0, 0, this.width, this.height);
                }
            },
            saveImageToAry: function () {
                this.cancelIndex = 0;
                let dataUrl = this.canvas.toDataURL();
                this.cancelList.push(dataUrl);
            },
            handleFileSelect(evt) {
                evt.stopPropagation();
                evt.preventDefault();
                const out = this;
                var files = evt.dataTransfer.files;

                for (var i = 0, f; f = files[i]; i++) {
                    var t = f.type ? f.type : 'n/a';
                    reader = new FileReader();


                    // 处理得到的图片
                    reader.onload = (function (theFile) {
                        return function (e) {
                            var image = new Image();
                            image.src = e.target.result;


                            image.onload = function () {
                                // 居中缩放
                                var hRatio;
                                var wRatio;
                                var left = 0;
                                var top = 0;
                                var maxWidth = out.width;
                                var maxHeight = out.height;
                                var Ratio = 1;
                                var width = image.width;
                                var height = image.height;
                                wRatio = maxWidth / width;
                                hRatio = maxHeight / height;
                                if (wRatio < 1 || hRatio < 1) {
                                    Ratio = (wRatio <= hRatio ? wRatio : hRatio);
                                }
                                // 根据比例重新设置图像大小
                                if (Ratio < 1) {
                                    width = width * Ratio;
                                    height = height * Ratio;

                                }
                                // 图片居中摆放
                                left = (maxWidth - width) / 2;
                                top = (maxHeight - height) / 2;

                                paint.drawImage(image, 0, 0, image.width, image.height, left, top, width, height);
                            }

                        };
                    })(f)
                    reader.readAsDataURL(f);
                }
            },
            onShape(){
                if(this.shape){
                    this.startShape(this.shape)
                }else {
                    this.cleanShape()
                }
            },
            cleanShape(){

            },
            startShape(graphType){
                const out = this;
                $("#cover").css("z-index",20);

                // chooseImg(obj);
                var canDraw = false;
                let shapePaint = document.getElementById('cover').getContext('2d');
                var startX;
                var startY;

                //鼠标按下获取 开始xy开始画图
                var mousedown = function(e){
                    scroolTop = $(window).scrollTop();
                    scroolLeft = $(window).scrollLeft();
                    canvasTop = $(canvas).offset().top - scroolTop;
                    canvasLeft = $(canvas).offset().left - scroolLeft;

                    shapePaint.strokeStyle = "#666666"
                    shapePaint.lineWidth = 4;

                    startX = e.clientX - canvasLeft;
                    startY = e.clientY - canvasTop;
                    shapePaint.moveTo(startX ,startY );
                    canDraw = true;

                    if(graphType == out.STRING_LINE){
                        shapePaint.beginPath();
                    }else if(graphType == 'circle'){
                        shapePaint.beginPath();
                        shapePaint.moveTo(startX ,startY );
                        shapePaint.lineTo(startX +1 ,startY+1);
                        shapePaint.stroke();

                    }else if(graphType == 'rubber'){
                        shapePaint.clearRect(startX - size * 10 ,  startY - size * 10 , size * 20 , size * 20);
                    }
                    // 阻止点击时的cursor的变化，draw
                    e=e||window.event;
                    e.preventDefault();
                };

                //鼠标离开 把蒙版canvas的图片生成到canvas中
                var mouseup = function(e){
                    e=e||window.event;
                    canDraw = false;
                    var image = new Image();
                        image.src = $("#cover").toDataURL();
                        image.onload = function(){
                            shapePaint.drawImage(image , 0 ,0 , image.width , image.height , 0 ,0 , canvasWidth , canvasHeight);
                            clearContext();
                            out.saveImageToAry();
                        }
                };

                //选择功能按钮 修改样式
                function chooseImg(obj){
                    var imgAry  = $(".draw_controller li");
                    for(var i=0;i<imgAry.length;i++){
                        $(imgAry[i]).removeClass('active');
                        $(imgAry[i]).addClass('normal');
                    }
                    $(obj).removeClass("normal");
                    $(obj).addClass("active");
                }

                // 鼠标移动
                var  mousemove = function(e){
                    e=e||window.event;
                    var x = e.clientX   - canvasLeft;
                    var y = e.clientY  - canvasTop;
                    //方块  4条直线搞定
                    if(graphType === out.STRING_SQUARE){
                        if(canDraw){
                            shapePaint.beginPath();
                            clearContext();
                            shapePaint.moveTo(startX , startY);
                            shapePaint.lineTo(x  ,startY );
                            shapePaint.lineTo(x  ,y );
                            shapePaint.lineTo(startX  ,y );
                            shapePaint.lineTo(startX  ,startY );
                            shapePaint.stroke();
                        }
                        //直线
                    }else if(graphType ===out.STRING_LINE){
                        if(canDraw){
                            shapePaint.beginPath();
                            clearContext();
                            shapePaint.moveTo(startX , startY);
                            shapePaint.lineTo(x  ,y );
                            shapePaint.stroke();
                        }
                    }else if(graphType === out.STRING_CIRCLE){
                        clearContext();
                        if(canDraw){
                            // 鼠标点击移动时产生的圆
                            paint.beginPath();
                            var radii = Math.sqrt((startX - x) *  (startX - x)  + (startY - y) * (startY - y));
                            paint.arc(startX,startY,radii,0,Math.PI * 2,false);
                            paint.stroke();
                        }else{
                            paint.beginPath();
                            paint.arc(x,y,20,0,Math.PI * 2,false);
                            paint.stroke();
                        }
                        //涂鸦 未画得时候 出现一个小圆
                    }
                };

                var clearContext = function(){
                    shapePaint.clearRect(0,0,out.width,out.height);
                }


                $('#cover').unbind();
                $('#cover').bind('mousedown',mousedown);
                $('#cover').bind('mousemove',mousemove);
                $('#cover').bind('mouseup',mouseup);
            },
            handleDragOver(evt) {
                evt.stopPropagation();
                evt.preventDefault();
            }


        }
        ,
        mounted() {
            this.initializeCanvas()
            this.pen = this.STRING_HIGHLIGHTER;
            document.onkeydown = function (e) {
                let key = window.event.keyCode;
                if (key === 32) {
                    this.value1 = !this.value1;
                }
            };
            this.canvas = document.getElementById('glCanvas');

            this.link = this.canvas.toDataURL()
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

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
        width: 96px;
        height: 486px;
        text-align: center;
        background-repeat: no-repeat;
        background-image: url(../../public/ruler.png);
        background-color: rgba(0, 0, 0, 0.55);
        background-size: cover;
        border-radius: 10px;
    }

    canvas{
        position: absolute;
        z-index : 0;
        left:0px;
        top:0px;
        border: black 1px solid;
    }
</style>
