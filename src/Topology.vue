<template>
  <div>
    <el-row v-loading="globalLoading" element-loading-spinner="el-icon-loading" element-loading-text="拼命加载中"
            style="margin-top: 20px">
      <el-row>
        <span style="margin-left: 20px">
              布局方向：
              <el-radio-group v-model="rankdir" @change="changeRankDirection">
                <el-radio label="TB">从上到下</el-radio>
                <el-radio label="LR">从左到右</el-radio>
              </el-radio-group>
            </span>
      </el-row>
      <el-row v-loading="topologyLoading"
              element-loading-text="正在构建拓扑图，请稍等..."
              element-loading-spinner="el-icon-loading"
              element-loading-background="rgba(255,255,255,0.75)">
        <div ref="topology" id="container">
        </div>
      </el-row>
    </el-row>

    <!--  拓扑图抽屉  -->
    <div>
      <el-drawer
          :title="topologyDrawerTitle"
          size="50%"
          :visible.sync="drawer"
          :direction="'rtl'"
          :before-close="handleClose">
        <el-row style="padding: 0 10px">
          <div v-if="currentNode.endpointType === 'DUBBO'">
            <application :current-node="currentNode"></application>
          </div>
          <div v-if="currentNode.endpointType === 'MYSQL'">
            <mysql :current-node="currentNode"></mysql>
          </div>
          <div v-if="currentNode.endpointType === 'REDIS'">
            <redis :current-node="currentNode"></redis>
          </div>
          <div v-if="currentNode.endpointType === 'ROCKET_MQ'">
            <rocket-mq :current-node="currentNode"></rocket-mq>
          </div>
          <div v-if="currentNode.endpointType === 'RELATION'">
            <relation :current-node="currentNode"></relation>
          </div>
        </el-row>
      </el-drawer>
    </div>
  </div>
</template>

<script>
import G6 from '@antv/g6'
import RocketMq from "./components/rocketMq";
import Mysql from "./components/mysql";
import Redis from "./components/redis";
import Relation from "./components/relation";
import Application from "./components/application";

const {getLabelPosition, transform} = G6.Util

const mockData = require('./mock-data.json');

export default {
  name: 'Topology',

  components: {
    Application,
    RocketMq,
    Mysql,
    Redis,
    Relation
  },

  data() {
    return {
      graph: null,
      drawer: false,
      direction: 'rtl',
      currentNode: {
        id: '',
        endpointType: '',
        name: '',
        source: '',
        target: ''
      },
      selectedEndpointSide: 'SERVER',
      applicationMap: {},
      topologyLoading: false,
      globalLoading: false,

      topologyDrawerTitle: '详情',

      rankdir: 'TB'
    }
  },

  methods: {

    handleClose(done) {
      done();
    },

    clickNode(item) {
      let node = item._cfg.model;
      this.currentNode.id = node.id;
      this.currentNode.endpointType = node.endpointType;
      this.currentNode.name = node.label

      this.topologyDrawerTitle = '详情'
      this.drawer = true;
    },

    clickEdge(item) {
      let edge = item._cfg.model;
      if (edge.source.split("#")[0] !== 'APPLICATION' || edge.target.split("#")[0] !== 'APPLICATION') {
        this.$message.info("请选择DUBBO调用路径！")
        return;
      }
      this.currentNode.source = edge.source;
      this.currentNode.target = edge.target;
      this.currentNode.endpointType = 'RELATION';
      this.currentNode.name = this.applicationMap[edge.source.split("#")[1]] + " => " + this.applicationMap[edge.target.split("#")[1]]
      this.currentNode.type = 'DUBBO_2_DUBBO';
      this.topologyDrawerTitle = this.currentNode.name
      this.drawer = true;
    },

    handleTopology(res, graph) {
      res.result.nodes.forEach(node => {
        node.icon = {show: true, width: 45, height: 45}
        if (node.endpointType === 'DUBBO') {
          node.icon.img = 'https://raw.githubusercontent.com/Tian90coder/ms-topo/cb46c024842f17462d15595fd2becc3729c6615b/public/dubbo.svg'
        } else if (node.endpointType === 'MYSQL') {
          node.icon.img = 'https://raw.githubusercontent.com/Tian90coder/ms-topo/cb46c024842f17462d15595fd2becc3729c6615b/public/mysql.svg'
        } else if (node.endpointType === 'REDIS') {
          node.icon.img = 'https://raw.githubusercontent.com/Tian90coder/ms-topo/cb46c024842f17462d15595fd2becc3729c6615b/public/redis.svg'
        } else if (node.endpointType === 'ROCKET_MQ') {
          node.icon.img = 'https://raw.githubusercontent.com/Tian90coder/ms-topo/cb46c024842f17462d15595fd2becc3729c6615b/public/rocket-mq.svg'
        }

        // 高亮节点
        // if (true) {
        //   node.color = '#40a9ff'
        //   node.type = 'background-animate'
        // }
      })
      graph.on('afterlayout', () => {
      })

      graph.removeBehaviors('activate-relations', 'default');
      graph.updateLayout(
          {
            type: 'dagre',
            ranksep: 60,
            rankdir: this.rankdir,
            nodesep: 20
          }
      )

      graph.on('node:mouseenter', (evt) => {
        const {item} = evt
        const edges = item.getEdges();
        edges.forEach((edge) => {
          edge.toFront();
          edge.getSource().toFront();
          edge.getTarget().toFront();
        });
        graph.paint();
        graph.setItemState(item, 'active', true)
      })
      graph.on('node:mouseleave', (evt) => {
        const edges = graph.getEdges();
        edges.forEach((edge) => {
          edge.toBack();
        });
        graph.paint();

        const {item} = evt
        graph.setItemState(item, 'active', false)
      })
      graph.on('node:click', (evt) => {
        const {item} = evt;
        this.clickNode(item);
      });
      // 点击时选中，再点击时取消
      graph.on('edge:click', (ev) => {
        const edge = ev.item;
        graph.setItemState(edge, 'selected', !edge.hasState('selected'));
        this.clickEdge(edge);
      });
      graph.on('edge:mouseenter', (ev) => {
        const edge = ev.item;
        graph.setItemState(edge, 'active', true);
        const source = edge.getSource();
        const target = edge.getTarget();
        edge.toFront();
        source.toFront();
        target.toFront();
        graph.paint();
      });

      graph.on('edge:mouseleave', (ev) => {
        const edge = ev.item;
        graph.setItemState(edge, 'active', false);

        const edges = graph.getEdges();
        edges.forEach((edge) => {
          edge.toBack();
        });
        graph.paint();
      });
      graph.read(res.result)
      this.topologyLoading = false;
    },

    async initTopologyGraph() {
      G6.registerNode(
          'background-animate',
          {
            afterDraw(cfg, group) {
              const r = cfg.size / 2;
              const back1 = group.addShape('circle', {
                zIndex: -3,
                attrs: {
                  x: 0,
                  y: 0,
                  r,
                  fill: cfg.color,
                  opacity: 0.6,
                },
                name: 'back1-shape',
              });
              const back2 = group.addShape('circle', {
                zIndex: -2,
                attrs: {
                  x: 0,
                  y: 0,
                  r,
                  fill: cfg.color,
                  opacity: 0.6,
                },
                name: 'back2-shape',
              });
              const back3 = group.addShape('circle', {
                zIndex: -1,
                attrs: {
                  x: 0,
                  y: 0,
                  r,
                  fill: cfg.color,
                  opacity: 0.6,
                },
                name: 'back3-shape',
              });
              group.sort();
              back1.animate(
                  {
                    r: r + 20,
                    opacity: 0.1,
                  },
                  {
                    duration: 3000,
                    easing: 'easeCubic',
                    delay: 0,
                    repeat: true,
                  },
              );
              back2.animate(
                  {
                    r: r + 20,
                    opacity: 0.1,
                  },
                  {
                    duration: 3000,
                    easing: 'easeCubic',
                    delay: 1000,
                    repeat: true,
                  },
              ); // 1s delay
              back3.animate(
                  {
                    r: r + 20,
                    opacity: 0.1,
                  },
                  {
                    duration: 3000,
                    easing: 'easeCubic',
                    delay: 2000,
                    repeat: true,
                  },
              );
            },
          },
          'circle',
      );
      G6.registerEdge(
          'arrow-running',
          {
            afterDraw(cfg, group) {
              const shape = group.get('children')[0]
              const arrow = group.addShape('marker', {
                attrs: {
                  x: 16,
                  y: 0,
                  r: 8,
                  lineWidth: 2,
                  stroke: '#3370ff',
                  fill: '#fff',
                  symbol: (x, y) => {
                    return [
                      ['M', x - 6, y - 4],
                      ['L', x - 2, y],
                      ['L', x - 6, y + 4]
                    ]
                  }
                }
              })

              arrow.animate(
                  (ratio) => {
                    const tmpPoint = shape.getPoint(ratio)
                    const pos = getLabelPosition(shape, ratio)
                    let matrix = [1, 0, 0, 0, 1, 0, 0, 0, 1]
                    matrix = transform(matrix, [
                      ['t', -tmpPoint.x, -tmpPoint.y],
                      ['r', pos.angle],
                      ['t', tmpPoint.x, tmpPoint.y]
                    ])
                    return {
                      x: tmpPoint.x,
                      y: tmpPoint.y,
                      matrix
                    }
                  },
                  {
                    repeat: true,
                    duration: 2000
                  }
              )
            },
            setState(name, value, item) {
              const group = item.getContainer();
              const shape = group.get('children')[0];
              if (name === 'active') {
                if (value) {
                  shape.attr('stroke', '#5394ef');
                } else {
                  shape.attr('stroke', '#C2C8D5');
                }
              }
              if (name === 'selected') {
                if (value) {
                  shape.attr('lineWidth', 2);
                } else {
                  shape.attr('lineWidth', 2);
                }
              }
            },
          },
          'quadratic'
      )
      const container = this.$refs.topology
      const width = 2200
      const height = 1100
      const graph = new G6.Graph({
        groupByTypes: false,
        container: 'container',
        width,
        height,
        fitCenter: true,
        modes: {
          default: [
            'drag-canvas',
            'zoom-canvas',
            'click-select',
            'drag-node'
          ]
        },
        layout: {
          type: 'dagre',
          ranksep: 60,
          rankdir: this.rankdir,
          nodesep: 20
        },
        defaultNode: {
          type: 'circle',
          size: 60,
          labelCfg: {
            position: 'bottom'
          }
        },
        defaultEdge: {
          type: 'arrow-running',
          style: {
            endArrow: true,
            lineWidth: 2,
            lineAppendWidth: 15,
            stroke: '#C2C8D5'
          }
        },
        nodeStateStyles: {
          selected: {
            stroke: '#5394ef'
          }
        },
        fitView: true
      })
      this.graph = graph
      this.topologyLoading = true;
      this.handleTopology(mockData, graph)
      if (typeof window !== 'undefined') {
        window.onresize = () => {
          if (!graph || graph.get('destroyed')) return
          if (!container || !container.scrollWidth || !container.scrollHeight) return
          graph.changeSize(container.scrollWidth, container.scrollHeight)
        }
      }
    },

    changeRankDirection(direction) {
      if (this.graph) {
        this.graph.updateLayout(
            {
              type: 'dagre',
              ranksep: 60,
              rankdir: direction,
              nodesep: 20
            }
        )
      }
    }
  },

  mounted: function () {
    this.initTopologyGraph();
  },
}
</script>

<style>
</style>