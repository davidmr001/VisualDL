<template>
  <div class="visual-dl-graph-charts">
    <svg
      class="visual-dl-page-left"
      ref="graphSvg">
      <g/>
    </svg>
  </div>
</template>
<script>
// service
import {getPluginGraphsGraph} from '../../service';

// The name 'svgToPngDownloadHelper' is just a placeholder.
// Loading the JS lib file will bind saveSvgAsPng to window.
import * as svgToPngDownloadHelper from './svgToPngDownloadHelper.js';

// for d3 drawing
import * as d3 from 'd3';

import has from 'lodash/has';

export default {
  props: {
    'doDownload': {
      type: Boolean,
      required: true,
    },
  },
  computed: {},
  data() {
    return {
    };
  },
  watch: {
    doDownload: function(val) {
      if (this.doDownload) {
        let svg = this.$refs.graphSvg;
        saveSvgAsPng(svg, "graph.png", {scale: 1.0});
        this.$emit('triggerDownload', false);
      }
    },
  },

  mounted() {
    let chartScope = this;
    getPluginGraphsGraph().then(({errno, data}) => {
      if (has(data, 'data') === false) {
        return;
      }

      let graphData = data.data;

      // d3 svg drawing
      let g = new dagreD3.graphlib.Graph()
        .setGraph({})
        .setDefaultEdgeLabel(function() {
          return {};
      });

      // eslint-disable-next-line
      let render = new dagreD3.render();
      let nodeKeys = [];

      let buildInputNodeLabel = function(inputNode) {
        // TODO(daming-lu): need more complex compound node
        let nodeLabel = 'id: ' + inputNode['name'] + '\n'
          + 'type: ' + inputNode['data_type'] + '\n'
          + 'dims: ' + inputNode['shape'].join(' x ');
        return nodeLabel;
      };

      // add input nodes
      if (has(graphData, 'input') === false) {
        return;
      }
      for (let i=0; i<graphData['input'].length; ++i) {
        let curInputNode = graphData['input'][i];
        let nodeKey = curInputNode['name'];
        g.setNode(
          nodeKey,
          {
            label: buildInputNodeLabel(curInputNode),
            style: 'opacity: 0.1; ' +
              'stroke-width: 3px; ' +
              'stroke: #333; ' +
              'stroke-color: #41b3a3; ' +
              'fill: #6c648b; ',
            class: 'input',
            labelStyle: 'font-size: 0.8em;',
          }
        );
        nodeKeys.push(nodeKey);
      }

      // add operator nodes then add edges from inputs to operator and from operator to output
      if (has(graphData, 'node') === false) {
        return;
      }
      for (let i=0; i<graphData['node'].length; ++i) {
        let curOperatorNode = graphData['node'][i];
        let nodeKey = 'opNode_' + i;

        // add operator node
        let curOpLabel = curOperatorNode['opType'];
        g.setNode(
          nodeKey,
          {
            label: curOpLabel + ' '.repeat(Math.floor(curOpLabel.length/5)),
            shape: 'rect',
            class: 'operator',
            style: 'stroke-width: 3px; ' +
              'opacity: 0.1; ' +
              'rx: 10; ' +
              'ry: 10; ' +
              'stroke: #333; ' +
              'stroke-color: #41b3a3; ' +
              'fill: #008c99; ',
          }
        );
        nodeKeys.push(nodeKey);

        // add output node
        if (has(graphData, 'output') === false) {
          return;
        }
        let outputNodeKey = curOperatorNode['output'][0];
        let outputPadding = ' '.repeat(Math.floor(outputNodeKey.length/2));
        g.setNode(
          outputNodeKey,
          {
            label: outputNodeKey + outputPadding,
            class: 'output',
            style: 'opacity: 0.1;' +
              'stroke-width: 3px; ' +
              'stroke-dasharray: 5, 5;' +
              'stroke: #333；' +
              'stroke-color: #41b3a3; ' +
              'fill: #015249; ',
            shape: 'diamond',

          }
        );
        nodeKeys.push(outputNodeKey);

        // add edges from inputs to node and from node to output
        for (let e=0; e<curOperatorNode['input'].length; ++e) {
          // TODO(daming-lu): hard-coding style here to this polyline shows shadows.
          g.setEdge(curOperatorNode['input'][e], nodeKey);
        }

        g.setEdge(nodeKey, curOperatorNode['output'][0], {
          style: 'stroke: #333;stroke-width: 1.5px',
        });
      }

      // TODO(daming-lu): add prettier styles to diff nodes
      let svg = d3.select('svg')
        .attr('font-family', 'sans-serif')
        .attr('font-size', '28px');

      render(d3.select('svg g'), g);

      // adjust viewBox so that the whole graph can be shown, with scroll bar
      svg.attr('viewBox', '0 0 ' + g.graph().width + ' ' + g.graph().height);

      svg.selectAll('.node').on('click', function(d, i) {
        let curNode = g.node(d);
        let nodeType = curNode.class;
        let nodeInfo = null;
        if (nodeType === 'operator') {
          let opIndex = d.slice(7); // remove prefix "opNode_"
          nodeInfo = graphData.node[opIndex];
        } else if (nodeType === 'input') {
          nodeInfo = graphData.input[d-1];
        } else {
          nodeInfo = 'output';
        }

        chartScope.$emit('curNodeUpdated',
                         {
                           'nodeType': nodeType,
                           'nodeInfo': nodeInfo,
        });
      });
    });
  },

  methods: {},
};
</script>
<style lang="stylus">
    .node
        cursor: pointer
    .edgePath path.path
        stroke: #333
        stroke-width: 1.5px

    .visual-dl-graph-charts
        width inherit
        margin 0 auto
        margin-bottom 20px

        .visual-dl-chart-box
            height 600px

        .visual-dl-img-box
            border solid 1px #e4e4e4
            position relative
            background #f0f0f0
            overflow hidden

            img
                box-sizing border-box
                margin 0 auto
                display block

            .small-img-box
                width 30px
                box-sizing border-box
                position absolute
                right 0
                top 0
                border-left solid 1px #e4e4e4
                border-bottom solid 1px #e4e4e4
                background #f0f0f0
                z-index 1

                .screen-handler
                    box-sizing border-box
                    position absolute
                    width 30px
                    height 20px
                    border solid 1px #666
                    top 0
                    left -1px
</style>
