<template>
    <div class="mapDiv">
        <div ref="mapDiv" id="mapDiv" class="mapDiv"></div>
        <div id="layerControl" class="layerControl">
            <div class="title"><label>图层列表</label></div>
            <ul ref="layerTree" id="layerTree" class="layerTree"></ul>
        </div>
        <MapTool></MapTool>
        <Modal v-model='selectpdf' title="pdf选项" @on-ok='exportPdf' @on-cancel='cancelPdf' width="600">
            <Form>
                <Row>
                   
                    <FormItem label='尺寸'>
                        <Select v-model='a1'>
                            <Option v-for='item in pageSize' :value='item.value' :key='item.value'>{{item.name}}</Option>
                        </Select>
                    </FormItem>
                   
                </Row>
                <Row>
                    <FormItem label='分辨率'>
                        <Select v-model='res'>
                            <Option v-for='item in resolution' :value='item.value' :key='item.value'>{{item.name}}</Option>
                        </Select>
                    </FormItem>
                </Row>
            </Form>
        </Modal>
    </div>
</template>
<script>
import"ol/ol.css"

import Map from "ol/Map"
import View from "ol/View"
import TileLayer from "ol/layer/Tile"
import XYZ from "ol/source/XYZ"
import TileWms from "ol/source/TileWMS"
import {defaults,ZoomToExtent,MousePosition,OverviewMap,ScaleLine} from "ol/control"
import {createStringXY} from "ol/coordinate"
import MapTool from "../views/MapTool"
import {getRenderPixel} from 'ol/render'
import Feature from 'ol/Feature'
import jsPDF from 'jspdf'
import {Point,LineString,Polygon,Circle as CirCle } from 'ol/geom'
import {fromCircle,fromExtent} from 'ol/geom/Polygon'
import geomCirCle from 'ol/geom/Circle'
import VectorLayer from 'ol/layer/Vector'
import VectorSource from 'ol/source/Vector'
import {Style,Fill,Stroke,Circle} from 'ol/style'
import {Draw, Modify, Snap,Interaction} from 'ol/interaction'
import {createRegularPolygon,createBox} from 'ol/interaction/Draw'
import bus from "@/utils/bus"
export default {
    name:'selectMap',
    components: {
        MapTool
    },
    data(){
        return{
            layer:[],
            layerName:[],
            type:'',
            draw:null,
            snap:null,
            layerVisibility:[],
            map:null,
            modify:null,
            tdtMap_vec:null,
            tdtMap_img:null,
            drawSource:null,
            drawLayer:null,
            vecLayerXY:null,
            a1:'a4',
            selectpdf:false,
            pageSize:[
                {
                    name:'A1',
                    value:'a1'
                },
                {
                    name:'A2',
                    value:'a2'
                },
                {
                    name:'A3',
                    value:'a3'
                },
                {
                    name:'A4',
                    value:'a4'
                },
                {
                    name:'A5',
                    value:'a5'
                },
            ],
            dims:{
                a0: [1189, 841],
                a1: [841, 594],
                a2: [594, 420],
                a3: [420, 297],
                a4: [297, 210],
                a5: [210, 148],
            },
            res:'72',
            resolution:[
                {
                    value:'72',
                    name:'72 dpi (fast)'
                },
                {
                    value:'150',
                    name:'150 dpi'
                },
                {
                    value:'300',
                    name:'300 dpi (slow)'
                }
            ]
        }
    },
    created(){
        bus.$on('mapHander',(type)=>{
           var  vm=this
            let map=this.map
             let view=this.map.getView();
             let mapDiv=this.$refs.mapDiv
              function mouseEvent(event){
                let map=vm.map
                mousePosition=map.getEventPixel(event)
                map.render()
            }
                   
            
            switch(type){
                case 'fangda':
                    view.setZoom(view.getZoom()+1)
                    break;
                case 'suoxiao':
                    view.setZoom(view.getZoom()-1)
                    break;
                case 'extent':
                    let extent=view.calculateExtent(this.map.getSize())
                    alert('当前范围'+extent[0]+","+extent[1]+","+extent[2]+","+extent[3])
                    break;
                case 'png':
                    /**
                     * 
                     */
                    //或者使用postcompose
                   this.map.once('rendercomplete',(event)=>{
                      let mapCanvas=document.createElement('canvas')
                      let size=this.map.getSize()
                      mapCanvas.width=size[0]
                      mapCanvas.height=size[1]
                      let mapContext=mapCanvas.getContext('2d')
                      Array.prototype.forEach.call(
                          document.querySelectorAll('.ol-layer canvas'),(canvas)=>{
                              if(canvas.width>0){
                                   let opacity=canvas.parentNode.style.opacity
                                    mapContext.globalAlpha=opacity===''?1:Number(opacity)
                                    let transform=canvas.style.transform
                                    let matrix=transform.match(/^matrix\(([^\(]*)\)$/)[1].split(',').map(Number)

                                    CanvasRenderingContext2D.prototype.setTransform.apply(mapContext,matrix)
                                    mapContext.drawImage(canvas,0,0)
                              }
                             
                          })
                        //   if (window.navigator.msSaveOrOpenBlob) {
                        //         const bstr = atob(imgUrl.split(',')[1])
                        //         let n = bstr.length
                        //         const u8arr = new Uint8Array(n)
                        //         while (n--) {
                        //         u8arr[n] = bstr.charCodeAt(n)
                        //         }
                        //         const blob = new Blob([u8arr])
                        //         window.navigator.msSaveOrOpenBlob(blob, 'chart-download' + '.' + 'png')
                        //     } else {
                        //         // 这里就按照chrome等新版浏览器来处理
                        //         const a = document.createElement('a')
                        //         a.href = mapCanvas.toDataURL()
                        //         a.setAttribute('download', 'chart-download')
                        //         a.click()
                        //     }
                          if(navigator.msSaveBlob){
                              navigator.msSaveBlob(mapCanvas.msToBlob(),'map.png')
                          }else{
                               const a = document.createElement('a')
                                a.href = mapCanvas.toDataURL()
                                a.setAttribute('download', 'map')
                                a.click()
                          }
                    })
                    this.map.renderSync()
                    
                    break;
                case 'pdf':
                    this.selectpdf=true
                   
                    break;
                case 'watch':
                     this.tdtMap_img.setVisible(true)
                    
                    //探查半径
                    let radius=75
                    document.addEventListener('keydown', function (evt) {
                        if (evt.which === 38) {
                            radius = Math.min(radius + 5, 150);
                            map.render();
                            evt.preventDefault();
                        } else if (evt.which === 40) {
                            radius = Math.max(radius - 5, 25);
                            map.render();
                            evt.preventDefault();
                        }
                    });
                    let mousePosition=null
                    mapDiv.addEventListener('mousemove',(event)=>{
                        
                        mousePosition=this.map.getEventPixel(event)
                        map.render()
                    })
                    //   mapDiv.addEventListener('mousemove',mouseEvent(event,vm))
                    
                    mapDiv.addEventListener('mouseout',()=>{
                        mousePosition=null
                        map.render()
                    })
                   

                    this.tdtMap_img.on('prerender',(event)=>{
                        let ctx=event.context
                        let pixelRadio=event.frameState.pixelRadio
                        ctx.save()
                        ctx.beginPath()
                        if(mousePosition){
                            let pixel=getRenderPixel(event,mousePosition)
                            let offset=getRenderPixel(event,[mousePosition[0]+radius,mousePosition[1]])
                            let canvasRadius=Math.sqrt(Math.pow(offset[0]-pixel[0],2)+Math.pow(offset[1]-pixel[1],2))
                            ctx.arc(pixel[0],pixel[1],canvasRadius,0,2*Math.PI)
                            ctx.lineWidth=(5*canvasRadius)/radius

                            // ctx.arc(mousePosition[0]*pixelRadio,mousePosition[1]*pixelRadio,radius*pixelRadio,0,2*Math.PI)
                            // ctx.lineWidth=5*pixelRadio
                            ctx.strokeStyle='rgb(0,0,0,0,5)'
                            ctx.stroke()
                        }
                        ctx.clip()
                    })

                    this.tdtMap_img.on('postrender',(event)=>{
                        let ctx=event.context
                        ctx.restore()
                    })
                    break;
                case 'unwatch':
                    let c=mapDiv
                    
                    mapDiv.removeEventListener('mousemove',mouseEvent())
                     mapDiv.removeEventListener('mouseout',()=>{
                       mousePosition=null
                        map.render()
                    })
                    // this.tdtMap_img.un('prerender',this.renderCanvas())
                    //  this.tdtMap_img.un('postrender')

            }
            

        })
        bus.$on('mapPicHander',(type)=>{
            //  let vecLayerXY=new VectorLayer()
              /**
               * 画几何图形
               */
             
            //  let drawSource=new VectorSource()
            //  let drawLayer=new VectorLayer({
            //      source:drawSource,
            //      style:new Style({
            //          fill: new Fill({
            //             color: 'rgba(255, 255, 255, 0.2)',
            //         }),
            //         stroke: new Stroke({
            //             color: 'black',
            //             width: 2,
            //         }),
            //         image: new Circle({
            //             radius: 7,
            //             fill: new Fill({
            //                 color: 'black',
            //             })
            //         })
            //      })
            //  })
            //  let modify=new Modify({source:this.drawSource})
            //  this.map.addInteraction(modify)
            switch(type){
                case 'pointDraw':
                    //点
                    let point=this.pointXY()
                    
                    //线
                    let line=this.lineXY()

                    //圆
                    let cirCle=this.circleXY()

                    //矩形
                    let square1=this.rectangeXY()

                    //矩形
                    let Rectangle=this.rectangleXY()

                    //三角形
                    let triangle=this.triangleXY()

                    
                    let vecSource=new VectorSource({
                        features:[point,line,cirCle,square1,Rectangle,triangle]
                    })
                    this.vecLayerXY.setSource(vecSource)
                    break;
                case 'clearXY':
                    
                    this.vecLayerXY.getSource().clear()
                    break
                case 'drawPoint':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'Point'
                    })
                    break;
                case 'drawLine':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'LineString'
                    })
                    
                    break;
                case 'drawPolygon':
                     this.draw=new Draw({
                        source:this.drawSource,
                        type:'Polygon'
                    })
                    
                    break;
                case 'drawCircle':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'Circle'
                    })
                    break;
                case 'drawSquare':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'Circle',
                        geometryFunction:createRegularPolygon(4)
                    })
                    break;
                case 'drawBox':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'LineString',
                        geometryFunction:createBox()
                    })
                    break;
                case 'handLine':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'LineString',
                        freehand:true
                    })
                    break
                case 'handPolygon':
                    this.draw=new Draw({
                        source:this.drawSource,
                        type:'Polygon',
                        freehand:true
                    })
                    break
                case 'cleardraw':
                    this.map.removeInteraction(this.draw)
                    this.map.removeInteraction(this.snap)
                    this.drawLayer.getSource().clear()
                    break
                case 'closeDraw':
                    if(this.draw!=null&&this.snap!=null){
                        this.map.removeInteraction(this.draw)
                        this.map.removeInteraction(this.snap)
                    }
                    break
                case 'openUpdate':
                     this.modify.setActive(true)
                    this.map.addInteraction(this.modify)
                    break
                case 'closeUpdate':
                   
                    // this.map.removeInteraction(this.draw)
                    // this.map.removeInteraction(this.snap)
                    this.modify.setActive(false)
                    this.map.removeInteraction(this.modify)
                    break
                 
            }
            /**
             * 几何图形
             */
            if(this.draw!=null&&type!='cleardraw'&&type!='closeDraw'&&type!='closeUpdate'){
                this.map.addInteraction(this.draw)
                    this.snap=new Snap({
                        source:this.drawSource
                })
                this.map.addInteraction(this.snap)
                //  this.map.addLayer(this.drawLayer)
               
                
            }
            
            //  this.map.addLayer(this.vecLayerXY)
        })
         bus.$on('mapInfo',(type)=>{
             switch(type){
                case 'image':

                    break;
                case 'text':
                    
                    break;
                case 'union':
                    
                    break;
                case 'popup':
                    
                    break;
                case 'merrage':
                    
                    break;
            }
            
        })
    
    
    
    
    bus.$on('removeInteraction',(type)=>{
         if(this.draw!=null&&this.snap!=null&&this.modify!=null){
             this.map.removeInteraction(this.draw)
             this.map.removeInteraction(this.snap)
            }

    })
    },
      beforeDestroy() {
        bus.$off("mapHander")
        bus.$off("mapPicHander")
        bus.$off("mapInfo")
  },
    mounted(){
        this.initMap()
    },
    // watch:{
    //     type:function (newQuestion, oldQuestion) {
    //         debugger
           
    //          this.map.removeInteraction(this.draw)
    //         this.map.removeInteraction(this.snap)
    //         this.map.addInteraction()
    //     }
    // },
    methods: {
        initMap(){
        /*
         * 比例尺dd
         */
        var scaleLineControl= new ScaleLine({
            // className: 'my-scale-line',
            bar: true,
            // text: true,
            minWidth:70,
            units: "metric"
        })
        /**
         * 鼠标移动获取位置信息
         */
        var mousePositionControl = new MousePosition({
            //坐标格式
            coordinateFormat: createStringXY(4),
            //地图投影坐标系（若未设置则输出为默认投影坐标系下的坐标）
            projection: 'EPSG:2435',
            //坐标信息显示样式类名，默认是'ol-mouse-position'
            className: 'custom-mouse-position',
            //显示鼠标位置信息的目标容器
            // target: document.getElementById('mouse-position'),
            //未定义坐标的标记
            undefinedHTML: '&nbsp;'
        })
        var tdtMap_cia=new TileLayer({
            name:'天地图影像标记层',
            
            source:new XYZ({
                url: "http://t0.tianditu.com/DataServer?T=cia_w&x={x}&y={y}&l={z}&tk=42dca576db031641be0524ee977ddd04",//parent.TiandituKey()为天地图密钥,
                crossOrigin: 'anonymous',
                wrapX: false
            }),
             visible:false
        })
        this.tdtMap_vec=new TileLayer({
            name:'天地图矢量图层',
            source: new XYZ({
                url:"http://t0.tianditu.com/DataServer?T=vec_w&x={x}&y={y}&l={z}&tk=42dca576db031641be0524ee977ddd04",
                crossOrigin: 'anonymous',
                wrapX:false
            })
        })
        var tdtMap_cva=new TileLayer({
            name:'天地图矢量标记图层',
            source:new XYZ({
                url:'http://t0.tianditu.com/DataServer?T=cva_w&x={x}&y={y}&l={z}&tk=42dca576db031641be0524ee977ddd04',
                crossOrigin: 'anonymous',
                wrapX:false
            }),
            
        })
        this.tdtMap_img=new TileLayer({
            name:'天地图影像图层',
            source:new XYZ({
                url:'http://t0.tianditu.com/DataServer?T=img_w&x={x}&y={y}&l={z}&tk=42dca576db031641be0524ee977ddd04',
                crossOrigin: 'anonymous',
                wrapX:false
            }),
            visible:false
        })
             /**
         * 鹰眼
         */
        var overviewMap = new OverviewMap({
            className: 'ol-overviewmap ol-custom-overviewmap',
            layers: [new TileLayer({
                name:'天地图矢量标记图层',
                source:new XYZ({
                    url:'http://t0.tianditu.com/DataServer?T=cva_w&x={x}&y={y}&l={z}&tk=42dca576db031641be0524ee977ddd04',
                    wrapX:false
                }),
                visible:false
            })],
        })
        //实例化Map对象
        this.map= new Map({
            target:'mapDiv',
            layers:[this.tdtMap_vec,tdtMap_cva,this.tdtMap_img,tdtMap_cia],
            view:new View({
                //地图中心点
                center:[12606072.0, 2650934.0],
                // projection: 'EPSG:4326',
                zoom:6
            }),
            controls:defaults({
                attibuttonOptions:({
                    collapsible: true
                })
            }).extend([mousePositionControl,overviewMap])
        })
        this.map.addControl(scaleLineControl);
        this.loadLayersControl(this.map,'layerTree')
        /**
         * 图片点击事件
         */
        this.map.on('singleclick',(e)=>{

        })

        /**
         * 初始化绘画图层
         */

        //绘画图层
        this.drawSource=new VectorSource()
        this.drawLayer=new VectorLayer({
            source:this.drawSource,
            style:new Style({
                fill: new Fill({
                color: 'rgba(255, 255, 255, 0.2)',
            }),
            stroke: new Stroke({
                color: 'black',
                width: 2,
            }),
            image: new Circle({
                radius: 7,
                fill: new Fill({
                    color: 'black',
                })
            })
            })
        })
        //坐标图层
        this.vecLayerXY=new VectorLayer()
        this.map.addLayer(this.drawLayer)
        this.map.addLayer(this.vecLayerXY)
         /**
          * 修改拖拽图层--绘画图层
          */
        this.modify=new Modify({source:this.drawSource})
        this.map.addInteraction(this.modify)
        },
        /**
         * 加载图层列表数据
         * map地图数据
         * 图层列表容器ID
         */
        loadLayersControl(map,id){
            var treeContent=this.$refs.layerTree
            var layers=map.getLayers()
            for (let i = 0; i < layers.getLength(); i++) {
                //获取每个图层的名称，是否可见属性
                let item=layers.getArray()[i]
                let itemName=item.get('name')
                let itemVisibity=item.getVisible()
                //新增li的元素，用来承载图层项
                let eleLi=document.createElement('li')
                eleLi.style="text-align: left;"
                //添加子节点
                treeContent.appendChild(eleLi)
                //创建复选框元素
                var eleInput=document.createElement('input')
                eleInput.type="checkbox"
                eleInput.name="layers"
                eleLi.append(eleInput)
                //创建label元素
                var eleLabel=document.createElement('label')
                eleLabel.className="layer"
                //设置图层名称
                this.setInnerText(eleLabel,itemName)
                eleLi.appendChild(eleLabel)
                //设置图层默认显示状态
                if(itemVisibity){
                    eleInput.checked=true
                }
                this.addChangeEvenr(eleInput,item)
            }
        },
        addChangeEvenr(ele,layer){
             ele.onclick=function () {
                if (ele.checked) {
                    layer.setVisible(true)
                }else{
                    layer.setVisible(false)
                }
            }

        },
        setInnerText(ele, text){
           if (typeof ele.treeContent=="string") {
               ele.textContent=text
           }else{
               ele.innerText=text
           }
        },
        exportPdf(){
            
            let c=this.res
            let d=this.a1
            let dim=this.dims[this.a1]
            let width=Math.round((dim[0]*this.res)/25.4)
            let height=Math.round((dim[1]*this.res)/25.4)
            let size=this.map.getSize()
            let viewResolution=this.map.getView().getResolution()
            this.map.once('rendercomplete',()=>{
                let mapCanvas=document.createElement('canvas')
                mapCanvas.width=width
                mapCanvas.height=height
                let mapContext=mapCanvas.getContext('2d')
                Array.prototype.forEach.call(
                    document.querySelectorAll('.ol-layer canvas'),(canvas)=>{
                        if(canvas.width>0){
                            let opacity=canvas.parentNode.style.opacity
                            mapContext.globalAlpha=opacity===''?1:Number(opacity)
                            let transform=canvas.style.transform
                            let matrix=transform.match(/^matrix\(([^\(]*)\)$/)[1].split(',').map(Number)
                            CanvasRenderingContext2D.prototype.setTransform.apply(mapContext,matrix)
                            mapContext.drawImage(canvas,0,0)
                        }
                    }
                )
                let pdf=new jsPDF('landscape',undefined,this.a1)
                pdf.addImage(mapCanvas.toDataURL('image/jpeg'),'JPEG',0,0,dim[0],dim[1])
                pdf.save('map.pdf')
                this.map.setSize(size)
                this.map.getView().setResolution(viewResolution)

            })
            let printSize=[width,height]
            this.map.setSize(printSize)
            let scaling=Math.min(width/size[0],height/size[1])
            this.map.getView().setResolution(viewResolution/scaling)
            
        },
        cancelPdf(){
            this.selectpdf=false
        },
        pointXY(){
            let point=new Feature({
                        geometry:new Point([11505912.0, 4011415.0])
                })
            point.setStyle(new Style({
                //填充色
                fill:new Fill({
                    color:'rgba(255,255,0.2)'
                }),
                //边框
                stroke:new Stroke({
                    color:'#ffcc33',
                    width:2
                }),
                //形状
                image:new Circle({
                    radius:17,
                    fill:new Fill({
                        color:'#ffcc33'
                    })
                })
            }))
            return point;
        },
        lineXY(){
            let line=new Feature({
                    geometry:new LineString([[13047453.0, 3737873.0], [12606072.0, 2650934.0]])
                })
            line.setStyle(new Style({
                fill:new Fill({
                    color:'rgba(255,255,0.2)'
                }),
                stroke:new Stroke({
                    color:'#ffcc33',
                    width:5
                }),
                image:new Circle({
                    radius:7,
                    fill:new Fill({
                        color:'#ffcc33'
                    })
                })
            }))
            return line
        },
        circleXY(){
             let cirCle=new Feature({
                geometry:new geomCirCle([12606072.0, 2650934.0],100000)
            })
            cirCle.setStyle(new Style({
                fill:new Fill({
                    color:'rgba(255,255,255,0.5)'
                }),
                stroke:new Stroke({
                    color:'#ffcc33',
                    width:6
                }),
                image:new Circle({
                    radius:7,
                    fill:new Fill({
                        color:'#ffcc33'
                    })
                })
            }))
            return cirCle
        },
        rectangeXY(){
            let squreCircle=new CirCle([13047453.0, 3737873.0], 100000)
            let square=new Feature({
                geometry:new fromCircle(squreCircle,4,150)
            })
            square.setStyle(
                new Style({
                    fill:new Fill({
                        color:'rgba(255,255,255,0.8)'
                    }),
                    stroke:new Stroke({
                        color:'red',
                        width:2
                    }),
                    image:new Circle({
                        radius:7,
                        fill:new Fill({
                            color:'#ffcc33'
                        })
                    })
                })
                
            )
            return square

        },
        rectangleXY(){
            let rectangle=new Feature({
                geometry:new fromExtent([12606072.0, 2650934.0, 13047453.0, 3737873.0])
            })
            rectangle.setStyle(new Style({
                fill:new Fill({
                    color:'rgba(255,255,255,0.5)'
                }),
                stroke:new Stroke({
                    color:'green'
                }),
                image:new Circle({
                    radius:7,
                    fill:new Fill({
                        color:'#ffcc33'
                    })
                })
            }))
            return rectangle
        },
        triangleXY(){
            let triangle=new Feature({
                geometry:new Polygon([[[11505912.0, 4011415.0], [12689769.0, 5107216.0], [13002855.0, 3522218.0]]])
            })
            triangle.setStyle(new Style({
                fill:new Fill({
                    color:'rgba(255,255,255,0.5)'
                }),
                stroke:new Stroke({
                    color:'#ffcc33',
                    width:2
                }),
                image:new Circle({
                    radius:7,
                    fill:new Fill({
                        color:'#ffcc33'
                    })
                })
            }))
            return triangle

        }
        

    }

}
</script>
<style lang="less" >

.mapDiv{
    width: 100%;
    height: 100%;
}
/** */
.toolDiv{
    position: absolute;
    left: 0px;
    top: 5px;
}

/* 图层控件层样式设置 */
.layerControl {
    position: absolute;
    bottom: 5px;
    min-width: 200px;
    max-height: 200px;
    right: 0px;
    top: 5px;
    /*在地图容器中的层，要设置z-index的值让其显示在地图上层*/
    z-index: 2001;
    color: #ffffff;
    background-color: #4c4e5a;
    /*边缘的宽度*/
    border-width: 10px;
    /*圆角的大小 */
    border-radius: 10px;
    /*边框颜色*/
    border-color: #000 #000 #000 #000;
}

.layerControl .title {
    text-align: left;
    font-weight: bold;
    font-size: 15px;
    margin: 10px;
}

.layerTree li {
    text-align: left;
    list-style: none;
    margin: 5px 10px;
}
.ol-scale-step-text {
    position: absolute;
    bottom: -5px;
    font-size: 12px;
    z-Index: 11;
    color: #000000;
}


/*=S 自定义鹰眼样式 */
        // .ol-overviewmap {
        //     right: 0.5em;
        //     bottom: 0.5em;
        // }
        .ol-overviewmap,.ol-custom-overviewmap, .ol-custom-overviewmap.ol-uncollapsible {
             bottom: 0;
             left: auto;
            /* 右侧显示 */
             right: 0;
            /* 顶部显示 */
             top: auto;
         
        }
        /* 鹰眼控件展开时控件外框的样式 */
        .ol-custom-overviewmap:not(.ol-collapsed)  {
            border: 1px solid black;
        }
         /* 鹰眼控件中地图容器样式 */
        .ol-custom-overviewmap .ol-overviewmap-map {
            border: none;
            width: 300px;
        }
        /* 鹰眼控件中显示当前窗口中主图区域的边框 */
        .ol-custom-overviewmap .ol-overviewmap-box {
            border: 2px solid red;
        }
        /* 鹰眼控件展开时其控件按钮图标的样式 */
        .ol-custom-overviewmap:not(.ol-collapsed) button{
            bottom: auto;
            left: auto;
            right: 1px;
            top: 1px;
        }
      
        /*=E 自定义鹰眼样式 */


</style>