<template>
  <div id="container"></div>
</template>

<script>
  import { defineComponent, onMounted } from 'vue';
  import G6 from '@antv/g6';
  import { treeData } from './tree';

  export default defineComponent({
    name: 'G6Mindmap',
    setup() {
      /**
       * format the string
       * @param {string} str The origin string
       * @param {number} maxWidth max width
       * @param {number} fontSize font size
       * @return {string} the processed result
       */
      const fittingString = (str, maxWidth, fontSize) => {
        const ellipsis = '...';
        const ellipsisLength = G6.Util.getTextSize(ellipsis, fontSize)[0];
        let currentWidth = 0;
        let res = str;
        const pattern = new RegExp('[\u4E00-\u9FA5]+'); // distinguish the Chinese charactors and letters
        str.split('').forEach((letter, i) => {
          if (currentWidth > maxWidth - ellipsisLength) return;
          if (pattern.test(letter)) {
            // Chinese charactors
            currentWidth += fontSize;
          } else {
            // get the width of single letter according to the fontSize
            currentWidth += G6.Util.getLetterWidth(letter, fontSize);
          }
          if (currentWidth > maxWidth - ellipsisLength) {
            res = `${str.substr(0, i)}${ellipsis}`;
          }
        });
        return res;
      };
      registerNode();
      registerEdge();
      onMounted(() => {
        drawGraph();
      });
      function registerNode() {
        G6.registerNode('rect-xml', {
          jsx: (cfg) => `
            <group>
              <rect style={{
                width: 200,
                height: 75,
              }}>
                <rect style={{
                  width: 150,
                  height: 20,
                  fill: ${cfg.color},
                  radius: [6, 6, 0, 0],
                  cursor: 'move'，
                  stroke: ${cfg.color}
                }} draggable="true">
                  <text style={{
                    marginTop: 2,
                    marginLeft: 75,
                    width: 150,
                    textAlign: 'center',
                    fontWeight: 'bold',
                    fill: '#fff' }}>${fittingString(cfg.description, 120, 14)}</text>
                </rect>
                <rect style={{
                  width: 150,
                  height: 55,
                  stroke: ${cfg.color},
                  fill: #ffffff,
                  radius: [0, 0, 6, 6],
                  overflow: 'hidden',
                  textOverflow: 'ellipsis',
                  whiteSpace: 'nowrap',
                  wordBreak: 'break-all',
                  maxWidth: 100,
                }}>
                  <text style={{ marginTop: 5, marginLeft: 3, fill: '#333', marginLeft: 4,width: 150, }}>描述: ${fittingString(cfg.description, 120, 14)}</text>
                  <text style={{ marginTop: 10, marginLeft: 3, fill: '#333', marginLeft: 4,width: 150, }}>创建者: ${fittingString(cfg.description, 120, 14)}</text>
                </rect>
              </rect>
              <circle style={{
                stroke: ${cfg.color},
                r: 10,
                fill: '#fff',
                marginLeft: 75,
                cursor: 'pointer'
              }} name="circle">
                <image name="img" style={{ img: 'https://gw.alipayobjects.com/zos/antfincdn/FLrTNDvlna/antv.png', width: 12, height: 12,  marginLeft: 69,  marginTop: -5 }} />
              </circle>
            </group>
          `,
          afterDraw: (cfg, group) => {
            console.log(group);
            const img = group.findAllByName('img');
            if (img[0]) {
              img[0].animate(
                (ratio) => {
                  return {
                    opacity: Math.abs(0.5 - ratio),
                  };
                },
                {
                  duration: 3000,
                  repeat: true,
                },
              );
            }
          },
        });
      }
      function registerEdge() {
        function randomColor() {
          const letters = '0123456789ABCDEF';
          let color = '#';
          for (let i = 0; i < 6; i++) {
            color += letters[Math.floor(Math.random() * 16)];
          }
          return color;
        }
        G6.registerEdge(
          'extra-shape-edge',
          {
            afterDraw(cfg, group) {
              // get the first shape in the graphics group of this edge, it is the path of the edge here
              // 获取图形组中的第一个图形，在这里就是边的路径图形
              const shape = group.get('children')[0];
              // get the coordinate of the mid point on the path
              shape.attr('stroke', randomColor()); // 随机颜色
              return shape;
            },
            update: undefined,
          },
          'cubic-horizontal',
        );
      }
      function setPosition(graph) {
        setTimeout(() => {
          const node = graph.findById('root');
          const canvasWidth = graph.get('width'); // 获取画布宽度
          const canvasHeight = graph.get('height'); // 获取画布高度
          const nodeWidth = node.getBBox().width; // 获取节点宽度
          const nodeHeight = node.getBBox().height; // 获取节点高度
          const nodeX = node.get('x'); // 获取节点 x 坐标

          // 计算节点应该移动到的新位置
          // const newX = (canvasWidth - nodeWidth) / 2;
          // const newY = 0 + nodeHeight / 2; // 假设最顶端的 Y 坐标为 0
          const range = getRange(graph);
          if (range) {
            const newX = nodeX;
            const newY = range.minY;
            moveNode(node, graph, newX, newY);
          }
        }, 1000);
      }
      function moveNode(node, graph, x, y) {
        // 更新节点位置
        node.updatePosition({ x: x, y: y });
        // 找到与该节点相连的边
        const edges = graph
          .getEdges()
          .filter((edge) => edge.getSource() === node || edge.getTarget() === node);

        // 遍历与该节点相连的边，更新它们的位置
        edges.forEach((edge) => {
          const sourceNode = edge.getSource();
          const targetNode = edge.getTarget();

          // 获取边的起点和终点坐标
          const sourcePosition = sourceNode.getModel();
          const targetPosition = targetNode.getModel();

          // 更新边的位置
          edge.update({
            source: { x: sourcePosition.x, y: sourcePosition.y },
            target: { x: targetPosition.x, y: targetPosition.y },
          });
        });
      }
      function genrateData(treeData, level = 0) {
        let data = treeData;
        data = Object.assign(data, {
          description: 'ant_type_name_' + data.id,
          label: 'Type / ReferType' + data.id,
          color: '#2196f3',
          level: level,
          id: level === 0 ? 'root' : data.id,
          meta: {
            creatorName: 'a_creator' + data.id,
          },
          type: 'rect-xml',
        });
        if (Array.isArray(data.children)) {
          Array.from(data.children, (item) => {
            return genrateData(item, level + 1);
          });
        }
        return data;
      }
      function getRange(graph) {
        const nodes = graph.getNodes();
        if (nodes.length === 0) return undefined;
        // 初始化 x 和 y 的最小值和最大值为第一个节点的坐标
        let minX = nodes[0].getModel().x;
        let maxX = nodes[0].getModel().x;
        let minY = nodes[0].getModel().y;
        let maxY = nodes[0].getModel().y;

        // 遍历所有节点，更新最小值和最大值
        nodes.forEach((node) => {
          const model = node.getModel();
          if (model.x < minX) minX = model.x;
          if (model.x > maxX) maxX = model.x;
          if (model.y < minY) minY = model.y;
          if (model.y > maxY) maxY = model.y;
        });
        return {
          minX,
          minY,
          maxX,
          maxY,
        };
      }
      function drawGraph() {
        const data = genrateData(treeData);
        const container = document.getElementById('container');
        const width = container.scrollWidth;
        const height = container.scrollHeight || 500;
        const graph = new G6.TreeGraph({
          container: 'container',
          width,
          height,
          modes: {
            default: [
              {
                type: 'collapse-expand',
                onChange: function onChange(item, collapsed) {
                  const data = item.get('model');
                  data.collapsed = collapsed;
                  return true;
                },
              },
              'drag-canvas',
              'zoom-canvas',
            ],
          },
          defaultNode: {
            size: 26,
            anchorPoints: [
              [0, 0.5],
              [1, 0.5],
            ],
          },
          defaultEdge: {
            // type: 'cubic-horizontal',
            type: 'extra-shape-edge',
          },
          layout: {
            type: 'mindmap',
            direction: 'H',
            // getHeight: () => {
            //   return 100;
            // },
            getWidth: () => {
              return 200;
            },
            getVGap: () => {
              return 30;
            },
            getHGap: () => {
              return 0;
            },
            getSide: (d) => {
              if (d.data.position === 'l') return 'left';
              return 'right';
            },
          },
        });

        // let centerX = 0;
        // graph.node(function (node) {
        //   if (node.id === 'Modeling Methods') {
        //     centerX = node.x;
        //   }

        //   return {
        //     label: node.id,
        //     labelCfg: {
        //       position:
        //         node.children && node.children.length > 0
        //           ? 'left'
        //           : node.x > centerX
        //             ? 'right'
        //             : 'left',
        //       offset: 5,
        //     },
        //   };
        // });

        graph.data(data);
        graph.render();
        setPosition(graph);
        graph.fitView();

        if (typeof window !== 'undefined')
          window.onresize = () => {
            if (!graph || graph.get('destroyed')) return;
            if (!container || !container.scrollWidth || !container.scrollHeight) return;
            graph.changeSize(container.scrollWidth, container.scrollHeight);
          };
      }
    },
  });
</script>

<style lang="less" scoped></style>
