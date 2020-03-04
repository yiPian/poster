<template>
    <view class="poster" :class="{'poster-display': isShowPoster && isDrawDone}">
        <canvas class="mycanvas" canvas-id="mycanvas" 
            :style="{
                width: canvasStyle.width + 'px', 
                height: canvasStyle.height + 'px'
            }"/>
        <img :src="imgSrc" :style="{
                width: canvasStyle.width + 'px', 
                height: canvasStyle.height + 'px',
                marginTop: canvasStyle.tb + 'px',
                marginLeft: canvasStyle.lr + 'px'
            }" class="animated zoomIn" v-if="isShowPoster && isDrawDone" />
        <view class="poster-btns" :class="{'animation-slide-bottom': isShowPoster && isDrawDone}" id="poster-btns">
            <view class="poster-btns-save" hover-class="hover" @tap="saveImage">保存至相册</view>
            <view hover-class="hover" @tap="hide">取消</view>
        </view>
    </view>
</template>
<script>
    /**
     * 发布信息详情海报的绘制
     */
    export default {
        props: {
            info: {
                type: Object
            }
        },
        data () {
            return {
                isShowPoster: false, // 是否显示绘制海报页面
                isDrawDone: false, // 是否绘制完成
                windowInfo: { // 可视尺寸
                    width: 0,
                    height: 0
                },
                canvasStyle: {
                    width: 200,
                    height: 200,
                    lr: 10, // 左右外距
                    tb: 10, // 上下外距
                    textDf: 14, // 默认字号
                    textSm: 12, // 小字号
                    padding: 10 // 内距
                },
                btnsHeight: 80, // 按钮层的高度
                codePath: '', // 小程序码本地的路
                imgList: [], // 暂存绘制图片的信息
                textList: [], // 暂存绘制文字的信息
                imgSrc: "" // 绘制完之后图片存放路径
            }
        },
        mounted () {
            let _ = this
            // 获取屏幕尺寸
            wx.getSystemInfo ({
                success (res) {
                    _.windowInfo.width = _.canvasStyle.width = res.windowWidth
                    _.windowInfo.height = res.windowHeight
                    // _.canvasStyle.width = _.windowInfo.width
                    // 根据像素比，计算文字的大小
                    _.canvasStyle.textDf *= res.pixelRatio / 2
                    _.canvasStyle.textSm *= res.pixelRatio / 2
                }
            })
            // 获得下面按钮层的高度
            let view = uni.createSelectorQuery().in(this).select("#poster-btns");
            view.fields({
                size: true
            }, data => {
                _.btnsHeight = data.height
            }).exec();
        },
        methods: {
            hide () {
                this.isShowPoster = false
            },
            /**
             * 开始绘制
             */
            show () {
                if (!this.isShowPoster) {
                    this.isShowPoster = true
                    if (!this.isDrawDone) {
                        uni.showLoading({
                            title: '生成中...'
                        })
                        //  获取小程序码
                        this.queryWxacode()
                    }
                }
            },
            /**
             * 获取小程序码
             */
            queryWxacode () {
                let that = this
                wx.cloud.callFunction ({
                    name: 'User',
                    data: {
                        $url: 'wxacode',
                        path: 'pages/index/infomation/index?id=' + this.info._id
                    }
                }).then(rs => {
                    if (rs.result.errCode > 0) {
                        this.hide()
                        this.$errTips()
                    } else {
                        //  这里返回的buffer 一会data一会没有
                        // 这里不要手动加base64的前缀
                        let base64 = wx.arrayBufferToBase64(rs.result.buffer.data || rs.result.buffer)
                        // 写入到本地
                        this.writeCode(base64)
                    }
                }).catch(err => {
                    this.hide()
                    this.$errTips()
                })
            },
            /**
             * 将小程序码写入到本地
             */
            writeCode (base64) {
                let _ = this
                let filePath = `${wx.env.USER_DATA_PATH}/${this.info._id}.jpeg`
                wx.getFileSystemManager().writeFile({
                    filePath,
                    data: base64,
                    encoding: 'base64',
                    success: () => {
                        _.codePath = filePath
                        // 计算画布高度
                        _.canvasHeight()
                    },
                    fail: err => {
                        uni.showModal({
                            title: '失败',
                            content: '请检查是否有生成图片的权限！',
                            showCancel: false
                        })
                    }
                })
            },
            /**
             * 计算绘制图片所需要的高度
             */
            canvasHeight () {
                let maxWidth = this.canvasStyle.width - 2 * this.canvasStyle.padding // 绘制可用宽度
                this.drawImgHeight(maxWidth).then((y) => {
                    this.canvasStyle.height = y
                    // 计算绘制文字的高度
                    this.drawTextHeight(maxWidth)
                    // 加上100的小区程序区域高度
                    this.canvasStyle.height += 100
                    // 开始绘制
                    this.draw(maxWidth)
                })
            },
            /**
             * 计算绘制图片所需要的高度
             */
            drawImgHeight (maxWidth) {
                return new Promise((resolve, reject) => {
                    let x, y
                    x = y = this.canvasStyle.padding
                    let length = this.info.images.length
                    if (length > 0) {
                        // 确保第一张图计算完之后，再计算后面的图片高度
                        this.getImgInfo(this.info.images[0], x, y, maxWidth).then(() => {
                            y += this.imgList[0].height + this.canvasStyle.padding
                            if (length > 1) {
                                let promiseArr = []
                                // 需要绘制的数量1～3 
                                let number = length > 3 ? 3 : (length - 1)
                                // 按等分计算大小
                                let size = Math.trunc ((maxWidth - this.canvasStyle.padding * (number - 1)) / number)
                                for (let i = 1; i <= number; i++) {
                                    x = (i - 1) * size + i * this.canvasStyle.padding
                                    promiseArr.push(
                                        this.getImgInfo(this.info.images[i], x, y, size, number > 1)
                                    )
                                }
                                Promise.all(promiseArr).then((res) => {
                                    // 更新画布的高度
                                    y = y + this.imgList[this.imgList.length - 1].height + this.canvasStyle.padding
                                    resolve(y)
                                }).catch(err => {
                                    reject(err)
                                })
                            } else {
                                resolve(y)
                            }
                        }).catch(err => {
                            reject(err)
                        })
                    } else {
                        resolve(y)
                    }
                })
            },
            /**
             * 获得图片的详情信息
             * @param url 图片的地址
             * @param maxWidth 图片的最大宽度
             * @param isSmall 是否是小图
             */
            getImgInfo (url, x , y, maxWidth, isSmall = false) {
                let _ = this
                return new Promise((resolve, reject) => {
                    wx.downloadFile({
                        url,
                        success (file) {
                            wx.getImageInfo({
                                src: file.tempFilePath,
                                success (res) {
                                    let height = maxWidth
                                    if (!isSmall) {
                                        height =  Math.trunc(maxWidth * res.height / res.width)
                                    }
                                    // 计算好位置、尺寸，暂存
                                    _.imgList.push({
                                        x,
                                        y,
                                        path: res.path,
                                        width: maxWidth,
                                        height
                                    })
                                    resolve()
                                }, 
                                fail (err) {
                                    reject(err)
                                }
                            })
                        },
                        fail (err) {
                            reject(err)
                        }
                    })
                })
            },
            /**
             * 计算绘制文字需要的高度
             */
            drawTextHeight (maxWidth) {
                let x = this.canvasStyle.padding
                let y = this.canvasStyle.height + this.canvasStyle.padding
                let size = this.canvasStyle.textDf
                // 按换行符 将内容分成多行
                let content = this.info.content.split('\n')
                for (let i = 0, len = content.length; i < len; i++) {
                    // 首尾去掉空格，内容空格保留一个
                    let text = content[i].replace(/(^\s*)|(\s*$)/g,"").replace(/\s+/g, ' ')
                    // 文字取出来一个个的算宽度
                    for (let j = 0, textLength = text.length; j < textLength; j++) {
                        this.textList.push({
                            x,
                            y,
                            color: '#333333',
                            size: this.canvasStyle.textDf,
                            text: text[j]
                        })
                        size = this.measureText(text[j], this.canvasStyle.textDf)
                        // 下一个字的x轴位置
                        x += size
                        // 超过最大宽，换行
                        if (x >= maxWidth && j < (textLength - 1)) {
                            x = this.canvasStyle.padding
                            y += size + this.canvasStyle.padding / 2
                        }
                    }
                    // 下一行的坐标
                    x = this.canvasStyle.padding
                    y += size + this.canvasStyle.padding
                }
                this.textList.push({
                    x,
                    y,
                    color: '#aaaaaa',
                    size: this.canvasStyle.textSm,
                    text: this.info.userInfo.nickName + ' 发布的「' + this.info.parent_title + '」'
                })
                // 更新canvas的高度
                this.canvasStyle.height = y + size + this.canvasStyle.padding 
            },
            draw (maxWidth) {
                let ctx = wx.createCanvasContext ('mycanvas', this)
                ctx.setFillStyle('white')
                ctx.fillRect(0, 0, this.canvasStyle.width, this.canvasStyle.height)
                ctx.draw()
                // 绘制图片
                for (let key in this.imgList) {
                    let val = this.imgList[key]
                    ctx.drawImage(val.path, val.x, val.y, val.width, val.height)
                    ctx.draw(true)
                }
                // 绘制文字
                for (let key in this.textList) {
                    let val = this.textList[key]
                    ctx.setFontSize(val.size)   //字号
                    ctx.setFillStyle(val.color)  //字体颜色
                    ctx.fillText(val.text, val.x, val.y)
                    ctx.draw(true)
                }
                // 绘制小程序区域
                let x = this.canvasStyle.padding
                let y = this.canvasStyle.height - this.canvasStyle.padding - 100
                // 绘制小程序码的底层
                ctx.setFillStyle('#fafafa')
                ctx.fillRect(x, y, maxWidth, 100)
                ctx.draw(true)
                // 绘制小程序码
                ctx.drawImage(this.codePath, x + 10, y + 10, 80, 80)
                ctx.draw(true)
                // 绘制文字
                x = 3 * x + 80
                y += 45
                ctx.setFillStyle('#333333')  //字体颜色
                ctx.setFontSize(this.canvasStyle.textDf)  //字号
                ctx.fillText('长安识别小程序码查看详情', x, y)
                ctx.setFillStyle('#aaaaaa')  //字体颜色
                ctx.setFontSize(this.canvasStyle.textSm)   //字号
                y += this.canvasStyle.textDf + this.canvasStyle.padding / 2
                ctx.fillText('来自新城名录', x, y)
                // 真机上transform 没有效果
                ctx.draw(true, setTimeout(() => {
                    wx.canvasToTempFilePath ({
                        fileType: 'jpg',
                        canvasId: 'mycanvas',
                        x: 0,
                        y: 0,
                        width: this.canvasStyle.width,
                        height: this.canvasStyle.height,
                        success: (res) => {
                            this.imgSrc = res.tempFilePath
                            // 计算canvas的缩放比例
                            this.countScale()
                            // 设置绘制完成
                            this.isDrawDone  = true
                            uni.hideLoading()
                            // 删除写入的数据
                            wx.getFileSystemManager().unlink({
                                filePath: this.codePath
                            })
                        }
                    }, this)
                }, 300))
            },
            countScale () {
                // 图片的最大宽高
                let maxWidth = this.windowInfo.width - 4 * this.canvasStyle.padding
                let maxheight = this.windowInfo.height -  4 * this.canvasStyle.padding - this.btnsHeight
                // 保证高度全部完整，宽度安比例缩小
                if (this.canvasStyle.height > maxheight) {
                    this.canvasStyle.width = Math.trunc(maxheight * this.canvasStyle.width / this.canvasStyle.height)
                    this.canvasStyle.height = maxheight
                } else {
                    this.canvasStyle.width = maxWidth
                    this.canvasStyle.height = Math.trunc(maxWidth * this.canvasStyle.height / this.canvasStyle.width)
                }
                // 计算间距
                this.canvasStyle.lr = (this.windowInfo.width - this.canvasStyle.width) / 2
                this.canvasStyle.tb = (this.windowInfo.height - this.btnsHeight - this.canvasStyle.height) / 2
                
            },
            /**
             * 计算单个字符的宽度
             */
            measureText (text, fontSize) {
                text = String(text);
                var text = text.split('');
                var width = 0;
                text.forEach(function(item) {
                    if (/[a-zA-Z]/.test(item)) {
                        width += 7;
                    } else if (/[0-9]/.test(item)) {
                        width += 5.5;
                    } else if (/\./.test(item)) {
                        width += 2.7;
                    } else if (/-/.test(item)) {
                        width += 3.25;
                    } else if (/[\u4e00-\u9fa5]/.test(item)) {  //中文匹配
                        width += 10;
                    } else if (/\(|\)/.test(item)) {
                        width += 3.73;
                    } else if (/\s/.test(item)) {
                        width += 2.5;
                    } else if (/%/.test(item)) {
                        width += 8;
                    } else {
                        width += 10;
                    }
                });
                return width * fontSize / 10;
            },

            /**
             * 保存图片到相册
             */
            saveImage () {
                let _ = this
                // 相册授权
                wx.getSetting({
                    success(res) {
                        // 进行授权检测，未授权则进行弹层授权
                        if (!res.authSetting['scope.writePhotosAlbum']) {
                            wx.authorize({
                                scope: 'scope.writePhotosAlbum',
                                success () {
                                    _.saveImageToPhotos()
                                },
                                // 拒绝授权时，则进入手机设置页面，可进行授权设置
                                fail(err) {
                                    wx.showModal({
                                        title: '提示',
                                        content: '需要您授权才能保存到相册',
                                        success: (res) => {
                                                if (res.confirm) {
                                                wx.openSetting({
                                                    success: function (data) {
                                                        console.log("openSetting success");
                                                    },
                                                    fail: function (data) {
                                                        console.log("openSetting fail");
                                                    }
                                                })
                                            }
                                        }
                                    })
                                }
                            })
                        } else {
                            // 已授权则直接进行保存图片
                            _.saveImageToPhotos()
                        }
                    },
                    fail(res) {
                        console.log(res);
                    }

                })
            },
            saveImageToPhotos () {
                wx.saveImageToPhotosAlbum({
                    filePath: this.imgSrc,
                    success: res => {
                        wx.showToast({
                            title: '保存成功',
                        });
                        this.hide()
                    },
                    fail (err) {
                        console.log('保存错误:%O', err)
                    }
                })
            }
        }
    }
</script>
<style>
    .poster {
        position: fixed;
        top: -9999px;
        left: -9999px;
        width: 100%;
        height: 100%;
        background-color: rgba(0, 0, 0, .5);
        z-index: 10;
    }
    .mycanvas {
        position: fixed;
        top: -9999px;
        left: -9999px;
    }
    .poster-display {
        top: 0;
        left: 0;
    }
    .poster-btns {
        position: absolute;
        left: 0;
        bottom: 0;
        width: 100%;
        background-color: #efefef;
        border-top-left-radius: 10rpx;
        border-top-right-radius: 10rpx;
        overflow: hidden;
    }
    .poster-btns view {
        height: 80rpx;
        line-height: 80rpx;
        text-align: center;
        color: #333;
        background-color: #fff;
    }
    .poster-btns-save {
        margin-bottom: 10rpx;
    }
</style>