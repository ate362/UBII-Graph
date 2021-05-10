<template>
  <div>
    <canvas id="canvas" class='litegraph' width='1024' height='720' style='border: 1px solid;'></canvas>
  </div>
</template>
<script>

import * as litegraph from "litegraph.js";
// import uuid4 from "uuid";
import "litegraph.js/css/litegraph.css";

export default {
  name: "LiteGraph",

  data() {
    return {
      outputContext: null,
      context: null,
      graph: null,
      lGraphCanvas: null
    };
  },

  mounted() {
    this.graph = new litegraph.LGraph();
    this.lGraphCanvas = new litegraph.LGraphCanvas('#canvas', this.graph);
    this.graph.start();
    
    const client = {
      id: "d8007d2a-4ea4-4e86-977e-96c7684f800a",
      name: "ubii-node-webbrowser VueJS Test",
      devices: [ 
        {
          id: "d0c43d91-f94b-4e45-8eaa-9c0b43642c8f",
          name: "web-example-mouse-pointer",
          components: [
            {topic: "/d8007d2a-4ea4-4e86-977e-96c7684f800a/web-example-mouse-pointer/mouse_client_position", ioType: 0, messageFormat: "ubii.dataStructure.Vector2"}, 
            {topic: "/d8007d2a-4ea4-4e86-977e-96c7684f800a/web-example-mouse-pointer/mirror_mouse", ioType: 0, messageFormat: "bool"}, 
            {topic: "/d8007d2a-4ea4-4e86-977e-96c7684f800a/web-example-mouse-pointer/mouse_server_position", ioType: 1, messageFormat: "ubii.dataStructure.Vector2"}
          ]
        }
      ]
    }

    client.devices.forEach(device => {
      this.registerClientNode(client.name, device, device.components)
    })
    

  },
  methods: {
    registerClientNode: function (clientName, dev, comp) {

      //node constructor class
      function node()
      {
        
        comp.forEach(c => {
          if (c.ioType == 0)
            this.addInput(c.topic, c.messageFormat)
          else
            this.addOutput(c.topic, c.messageFormat) 
        })
      }
      //this.properties = { precision: 1 };
      node.title = dev.name

      //function to call when the node is executed
      node.prototype.onExecute = function()
      {
        // var A = this.getInputData(0);
        // if( A === undefined )
        //   A = 0;
        // var B = this.getInputData(1);
        // if( B === undefined )
        //   B = 0;
        // this.setOutputData( 0, A + B );
      }

      //register in the system
      litegraph.LiteGraph.registerNodeType(clientName+"/"+dev.name, node)
    },
    addNode: function(id, device, components) {

      console.log(id, device, components)

      var node_const = litegraph.createNode("basic/const");
      node_const.pos = [200,200];
      this.graph.add(node_const);
      node_const.setValue(4.5);


    }
  }
}
</script>