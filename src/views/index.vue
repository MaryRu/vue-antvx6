<template>
  <div class="container_warp">
  <div id="containerChart"></div>
  <RightDrawer class="right_drawer" :drawerType="type" :selectCell="selectCell" :graph="graph" :grid="grid" @deleteNode="deleteNode"></RightDrawer>
  <div class="operating">
    <div class="btn-group">
      <div class="btn" title="圆形节点" @mousedown="startDrag('Circle',$event)">
        <i class="iconfont icon-circle"></i>
      </div>
      <div class="btn" title="正方形节点" @mousedown="startDrag('Rect',$event)">
        <i class="iconfont icon-square"></i>
      </div>
      <div class="btn" title="条件节点">
        <i class="iconfont icon-square rotate-square" @mousedown="startDrag('polygon',$event)"></i>
      </div>
      <div class="btn-group_tips" v-if="showTips">
        拖拽生成<br/>资产拓扑图形
      </div>
    </div>
    <div class="btn-group">
      <Tooltip content="直线箭头" placement="bottom">
          <div :class=" ['btn',currentArrow === 1?'currentArrow':'']" @click="changeEdgeType('normal')">
            <i class="iconfont icon-ai28"></i>
          </div>
      </Tooltip>
      <Tooltip content="曲线箭头" placement="bottom">
          <div :class=" ['btn',currentArrow === 2?'currentArrow':'']" @click="changeEdgeType('smooth')">
            <i class="iconfont icon-Down-Right"></i>
          </div>
      </Tooltip>
      <Tooltip content="直角箭头" placement="bottom">
          <div :class=" ['btn',currentArrow === 3?'currentArrow':'']" @click="changeEdgeType()">
            <i class="iconfont icon-jiantou"></i>
          </div>
      </Tooltip>
    </div>
    <div class="btn-group">
      <Tooltip content="删除" placement="bottom">
          <div class="btn" @click="deleteNode()" style="margin-top: 5px;">
            <i class="iconfont icon-shanchu"></i>
          </div>
      </Tooltip>
      <Tooltip content="保存PNG" placement="bottom">
          <div class="btn" @click="saveToPNG()" title="保存">
            <i class="iconfont icon-baocun"></i>
          </div>
      </Tooltip>
    </div>
  </div>
  </div>
</template>

<script>
import '@antv/x6-vue-shape'
import { Graph, Shape, Addon, FunctionExt, DataUri} from '@antv/x6';
import RightDrawer from './components/RightDrawer';
import insertCss from 'insert-css';
import {startDragToGraph} from './Graph/methods.js'
const data = {};
export default {
    data() {
      return {
        graph:'',
        value1: true,
        type:'grid',
        selectCell:'',
        connectEdgeType:{  //连线方式
          connector: 'normal',
          router: {
            name: ''
          }
        },
        showTips:false,
        currentArrow:1,
        grid:{ // 网格设置
          size: 20,      // 网格大小 10px
          visible: true, // 渲染网格背景
          type: 'mesh',
          args: {
            color: '#D0D0D0',
            thickness: 1,     // 网格线宽度/网格点大小
            factor: 10, 
          }
        }
      }
    },
    components:{
      RightDrawer
    },
    methods: {
      initX6(){
          var _that = this
          this.graph = new Graph({
              container: document.getElementById('containerChart'),
              width: 1700,
              height: '100%',
              grid: _that.grid,
              resizing: { //调整节点宽高
                enabled: true,
                orthogonal:false,
              }, 
              selecting: true, //可选
              snapline:  true,
              interacting: {
                edgeLabelMovable: true
              },
              connecting: { // 节点连接
                anchor: 'center',
                connectionPoint: 'anchor',
                allowBlank: false,
                snap: true,
                createEdge () {
                  return new Shape.Edge({
                    attrs: {
                      line: {
                        stroke: '#1890ff',
                        strokeWidth: 1,
                        targetMarker: {
                          name: 'classic',
                          size: 8
                        },
                        strokeDasharray: 0, //虚线
                        style: {
                          animation: 'ant-line 30s infinite linear',
                        }
                      }
                    },
                    label: {
                      text:''
                    },
                    connector: _that.connectEdgeType.connector,
                    router: {
                      name: _that.connectEdgeType.router.name || ''
                    },
                    zIndex: 0
                  })
                },
              },
              highlighting: {
                magnetAvailable: {
                  name: 'stroke',
                  args: {
                    padding: 4,
                    attrs: {
                      strokeWidth: 4,
                      stroke: '#6a6c8a'
                    }
                  }
                }
              },
          });
          insertCss(`
            @keyframes ant-line {
              to {
                  stroke-dashoffset: -1000
              }
            }
          `)
          this.graph.fromJSON(data)
          this.graph.history.redo()
          this.graph.history.undo()
          // 鼠标移入移出节点
          this.graph.on('node:mouseenter',FunctionExt.debounce(() => {
              const container =  document.getElementById('containerChart')
              const ports = container.querySelectorAll(
                '.x6-port-body'
              )
              this.showPorts(ports, true)
            }),
            500
          )
          this.graph.on('node:mouseleave', () => {
            const container =  document.getElementById('containerChart')
            const ports = container.querySelectorAll(
              '.x6-port-body'
            )
            this.showPorts(ports, false)
          })
          this.graph.on('blank:click', () => {
            this.type = 'grid'
          })
          this.graph.on('cell:click', ({ cell }) => {
            this.type = cell.isNode() ? 'node' : 'edge'
          })
          this.graph.on('selection:changed', (args) => {
            args.added.forEach(cell => {
              this.selectCell = cell
              if(cell.isEdge()){
                cell.isEdge() && cell.attr('line/strokeDasharray', 5) //虚线蚂蚁线
                cell.addTools([
                  {
                    name: 'vertices',
                    args: {
                      padding: 4,
                      attrs: {
                        strokeWidth: 0.1,
                        stroke: '#2d8cf0',
                        fill: '#ffffff',
                      }
                    },
                  },
                ])
              }
            })
            args.removed.forEach(cell => {
              cell.isEdge() && cell.attr('line/strokeDasharray', 0)  //正常线
              cell.removeTools()
            })
          })
      },
      showPorts (ports, show) {
        for (let i = 0, len = ports.length; i < len; i = i + 1) {
          ports[i].style.visibility = show ? 'visible' : 'hidden'
        }
      },
      // 拖拽生成正方形或者圆形
      startDrag(type,e){
        startDragToGraph(this.graph,type,e)
      },
      // 删除节点
      deleteNode(){
        const cell = this.graph.getSelectedCells()
        this.graph.removeCells(cell)
        this.type = 'grid'
      },
      // 保存png
      saveToPNG () {
        this.$nextTick(()=>{
          this.graph.toPNG((dataUri) => {
            // 下载
            DataUri.downloadDataUri(dataUri, '资产拓扑图.png')
          },{
            backgroundColor: 'white',
            padding: {
              top: 50,
              right: 50,
              bottom: 50,
              left: 50
            },
            quality: 1,
            copyStyles:false
          })
        })
      },
      // 改变边形状
      changeEdgeType(e){
        if(e === 'normal'){
          this.connectEdgeType = {
            connector: 'normal',
            router: {name: ''}
          }
          this.currentArrow = 1
        }else if (e === 'smooth'){
          this.connectEdgeType = {
            connector: 'smooth',
            router: {name: ''}
          }
          this.currentArrow = 2
        }else{
          this.connectEdgeType = {
            connector: 'normal',
            router: {name: 'manhattan'}
          }
          this.currentArrow = 3
        }
      }
    },
    mounted(){
      this.initX6()
      setTimeout(()=>{
        this.showTips = true
      },1000)
      setTimeout(()=>{
        this.showTips = false
      },5000)
    }
}
</script>
<style lang="less">
  @import '../assets/iconfont.css';
  @import './index.less';
</style>