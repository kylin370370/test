<template>
    <button @click="frameSelect" id="frameSelectButton">框选幕墙块</button>
    <button @click="clickSelect" id="clickSelectButton">单选幕墙块</button>
    <div>{{ content }}</div>
    <canvas id="canvas" width="4056" height="3040"></canvas>
</template>

<script>
import axios from 'axios';
var tag = [];
// 创建一个二维数组来存储
var _data = [];
var _id;
var nowx, nowy;
var isFrameSelect = null;

function mousePosition(ev) {
    if (ev.pageX || ev.pageY) {
        return { x: ev.pageX, y: ev.pageY };
    }
}
function mouseMove(ev) {
    ev = ev || window.event;
    var mousePos = mousePosition(ev);
    nowx = mousePos.x;
    nowy = mousePos.y;
}
document.onmousemove = mouseMove;
function Show(el) {
    if (!isFrameSelect) {
        var x = nowx - el.offsetLeft;
        var y = nowy - el.offsetTop;
        var id = _data[y][x];
        _id = id;
        id = "ID:" + id;
        x = "X:" + x;
        y = "Y:" + y;
        alert(x + "," + y + "\n" + id);
    }
}

window.onload = function () {
    var canvas = document.getElementById('canvas');
    var ctx = canvas.getContext('2d');

    // 从后端获取图片,并显示到画布中
    var img = new Image();
    axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 从后端获取图片数据
        .then(response => {
            var blob = response.data;
            img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
            img.onload = function () {
                canvas.width = img.width;
                canvas.height = img.height;
                ctx.drawImage(this, 0, 0);
            };
        })
        .catch(error => {
            console.error('Error:', error);
        });

    // 画布的点击事件,把同一个幕墙块涂黑
    canvas.onclick = function () {
        if (!isFrameSelect && isFrameSelect !== null) {
            Show(this);
            // 从后端获取图片数据
            axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 由后端获取图片数据
                .then(response => {
                    var blob = response.data;
                    var img = new Image();
                    img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
                    img.onload = function () {
                        ctx.drawImage(this, 0, 0);
                        if (_id != 0)
                            for (var i = 0; i < tag[_id].length; i++) {
                                ctx.fillStyle = 'rgba(0, 0, 0, 1)';
                                ctx.fillRect(tag[_id][i].x, tag[_id][i].y, 1, 1);
                            }
                    };
                })
                .catch(error => {
                    console.error('Error:', error);
                });
        }
    };

    // 下面是实现框选功能
    var rect = {}; // 用于存储矩形的坐标信息
    var oldx = 0;
    var oldy = 0;
    var oldw = 0;
    var oldh = 0;
    canvas.addEventListener("mousedown", function (event) {
        if (isFrameSelect) {
            var xiao = canvas.getBoundingClientRect(); // 获取canvas元素相对于视口的偏移量
            rect.startX = event.clientX - xiao.left; // 记录矩形起始点的 x 坐标
            rect.startY = event.clientY - xiao.top;  // 记录矩形起始点的 y 坐标
            axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 由后端获取图片数据
                .then(response => {
                    var blob = response.data;
                    var img = new Image();
                    img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
                    ctx.drawImage(img, 0, 0);
                })
                .catch(error => {
                    console.error('Error:', error);
                });

            canvas.addEventListener("mousemove", drawRect); // 监听鼠标移动事件，实时绘制矩形
            canvas.addEventListener("mouseup", getCornerPoints); // 监听鼠标松开事件，获取矩形的四个角点坐标
        }
    });

    function drawRect(event) {
        if (isFrameSelect) {
            // 实时计算矩形的宽度和高度
            var xiao = canvas.getBoundingClientRect();
            rect.width = (event.clientX - xiao.left) - rect.startX;
            rect.height = (event.clientY - xiao.top) - rect.startY;

            // 绘制矩
            if (!(oldx === rect.startX && oldy === rect.startY && oldw === rect.width && oldh === rect.height)) {
                axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 由后端获取图片数据
                    .then(response => {
                        var blob = response.data;
                        var img = new Image();
                        img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
                        img.onload = function () {
                            ctx.drawImage(img, 0, 0);
                            ctx.fillStyle = 'rgba(255, 0, 0, 0.5)';
                            ctx.fillRect(rect.startX, rect.startY, rect.width, rect.height);
                        }
                    })
                    .catch(error => {
                        console.error('Error:', error);
                    });

            }
        }
    }

    function getCornerPoints() {
        if (isFrameSelect) {
            axios.post('http://127.0.0.1:5000/coordinate', {
                'topLeft': { 'x': rect.startX, 'y': rect.startY },
                'topRight': { 'x': rect.startX + rect.width, 'y': rect.startY },
                'bottomLeft': { 'x': rect.startX, 'y': rect.startY + rect.height },
                'bottomRight': { 'x': rect.startX + rect.width, 'y': rect.startY + rect.height },
            })
                .then(response => {
                    console.log(response.data);
                })
                .catch(error => {
                    console.error('Error sending data:', error);
                });

            // 移除事件监听器
            canvas.removeEventListener("mousemove", drawRect);
            canvas.removeEventListener("mouseup", getCornerPoints);

            oldx = rect.startX;
            oldy = rect.startY;
            oldw = rect.width;
            oldh = rect.height;
            rect.width = 0;
            rect.height = 0;
        }
    }
};

export default {
    data() {
        return {
            content: "请选择选取幕墙块的方式"
        }
    },
    methods: {
        async readFile() {
            try {
                const response = await axios.get('\thttp://127.0.0.1:4523/m1/2993287-0-default/subwall_masks', { responseType: 'text' });
                return response.data;
            } catch (error) {
                console.error('Error:', error);
                return null;
            }
        },

        frameSelect() {
            isFrameSelect = true;
            this.content = "当前是框选幕墙块";
            var canvas = document.getElementById('canvas');
            var ctx = canvas.getContext('2d');

            // 从后端获取图片,并显示到画布中
            var img = new Image();
            axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 从后端获取图片数据
                .then(response => {
                    var blob = response.data;
                    img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
                    img.onload = function () {
                        canvas.width = img.width;
                        canvas.height = img.height;
                        ctx.drawImage(this, 0, 0);
                    };
                })
                .catch(error => {
                    console.error('Error:', error);
                });
        },

        clickSelect() {
            isFrameSelect = false;
            this.content = "当前是单击选取幕墙块";
            var canvas = document.getElementById('canvas');
            var ctx = canvas.getContext('2d');

            // 从后端获取图片,并显示到画布中
            var img = new Image();
            axios.get('http://127.0.0.1:5000/get_image', { responseType: 'blob' }) // 从后端获取图片数据
                .then(response => {
                    var blob = response.data;
                    img.src = URL.createObjectURL(blob); // 创建一个临时的 URL，用于指向获取到的图片数据
                    img.onload = function () {
                        canvas.width = img.width;
                        canvas.height = img.height;
                        ctx.drawImage(this, 0, 0);
                    };
                })
                .catch(error => {
                    console.error('Error:', error);
                });
        }
    },

    async created() {
        function pair() {
            const pair = {
                x: -1,
                y: -1
            };
            return pair;
        }

        try {
            // 同步机制,等待文本文件传来了才继续运行后面的代码,否则等待
            const fileData = await this.readFile();
            if (fileData) {
                this.title = fileData;

                var rows = this.title.split("\n");
                var mx = 0;
                // var cnt=0;
                for (var i = 0; i < rows.length - 1; i++) {
                    var nums = rows[i].split(" ");

                    // 创建一个子数组来
                    var row = [];

                    // 遍历数字
                    for (var j = 0; j < nums.length; j++) {
                        // 将数字转为整数并添加到子数组中
                        row.push(parseInt(nums[j]));
                        mx = Math.max(mx, parseInt(nums[j]));
                    }

                    // 将子数组添加到二维数组中
                    _data.push(row);
                }

                // 处理剩余的逻辑
                for (i = 0; i <= mx + 5; i++) {
                    nums = [];
                    tag.push(nums);
                }
                for (i = 0; i < _data.length; i++) {
                    for (j = 0; j < _data[0].length; j++) {
                        var x = _data[i][j];
                        var now = pair();
                        now.x = j;
                        now.y = i;
                        tag[x].push(now);
                    }
                }

                alert('共检测到' + mx + '块幕墙');
            }
        } catch (error) {
            console.error('Error processing file:', error);
        }
    }
}

</script>



<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 10px;
}
</style>

<style type="text/css">
* {
    padding: 0;
    margin: 0;
}

#frameSelectButton,
#clickSelectButton {
    font-size: 100px;
    margin-left: 10px;
    margin-right: 10px;
    margin-bottom: 20px;
}

div {
    margin-left: auto;
    margin-right: auto;
    margin-bottom: 20px;
    font-size: 90px;
}

body {
    text-align: center;
    padding-top: 20px;
}

canvas {
    display: block;
    box-shadow: 0 0 10px #333333;
    margin: 0 auto;
}
</style>
