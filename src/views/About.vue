<template>
  <div>
  <Error :alert="trigger" :msg="msg"/>
  <div class="header-editor">
    <span style="padding-right: 5px;">Sessions:</span>
    <treeselect v-model="selectedSessionId" @input="showSessionPipeline()" :load-options="loadOps" :options="availableSessions" :auto-load-root-options="false" :multiple="false" placeholder="Sessions..."/>
  </div>
    <canvas id="canvas" class='litegraph' width='1024' height='720' style='border: 1px solid;'></canvas>
  </div>
</template>
<script>

import * as litegraph from "litegraph.js";
// import uuid4 from "uuid";
import "litegraph.js/css/litegraph.css";

import UbiiClientService from '../ubiiNode/ubiiClientService'
import { DEFAULT_TOPICS } from '@tum-far/ubii-msg-formats';

import Error from '../components/Error.vue'

// import the component
import Treeselect from '@riophae/vue-treeselect'
import { LOAD_ROOT_OPTIONS } from '@riophae/vue-treeselect'
// import the styles
import '@riophae/vue-treeselect/dist/vue-treeselect.css'

export default {
  name: "LiteGraph",
  components: {
    Treeselect,
    Error
  },
  data() {
    return {
      outputContext: null,
      context: null,
      graph: null,
      lGraphCanvas: null,

      trigger: 0,
      msg: '',

      selectedSessionId: null,
      selectedSession: null,
      availableSessions: null,

      clientsOfInterest: null
    };
  },

  mounted() {
    this.graph = new litegraph.LGraph();
    this.lGraphCanvas = new litegraph.LGraphCanvas('#canvas', this.graph);
    this.graph.start();
    
    litegraph.LiteGraph.clearRegisteredTypes()
        
  },
  methods: {
    registerProcessNode: function(sname, proc, inp, out) {
      //node constructor class
      function node()
      {
        
        inp.forEach(i => {
          this.addInput(i.internalName, i.messageFormat)
        })
        out.forEach(o => {
          this.addOutput(o.internalName, o.messageFormat)
        })

      }
      node.title = proc.name

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

      litegraph.LiteGraph.registerNodeType("Sessions/"+sname+"/"+proc.name, node)

    },
    registerClientNode: function (clientName, dev, comp) {

      //node constructor class
      function node()
      {
        
        comp.forEach(c => {
          if (c.ioType == 1)
            this.addInput(c.topic.split("/").pop(), c.messageFormat)
          else
            this.addOutput(c.topic.split("/").pop(), c.messageFormat) 
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
      litegraph.LiteGraph.registerNodeType("Clients/"+clientName+"/"+dev.name, node)
    },
    addNode: function(name, pos) {

      var node_const = litegraph.LiteGraph.createNode(name);
      node_const.pos = pos;
      this.graph.add(node_const);
    },
    loadProcModules: async function () {
       const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
          {
            topic: DEFAULT_TOPICS.SERVICES.SESSION_RUNTIME_GET_LIST
          }
        ) : null

        const sList = res?.sessionList ?? {}
        // console.log(sList)
        try { 
          this.selectedSession = sList.filter(val => val.status === 1 && this.selectedSessionId == val.id)[0]
        } catch {
          this.msg = 'Selected Session was not found.'
          this.trigger= !this.trigger
        }
    },
    registerSessionNodes: async function () {
      this.selectedSession.processingModules.forEach(proc => {
        this.registerProcessNode(this.selectedSession.name, proc, proc.inputs, proc.outputs)
      })
    },
    loadComponents: async function () {
       
      const clientsDevices = await this.selectedSession.ioMappings.map(val => {
        const inputArray = val.inputMappings.map(val => {
          const sp = val.topic.split('/')
          return sp[1]+'.'+sp[2]
        })
        const outputArray = val.outputMappings.map(val => {
          const sp = val.topic.split('/')
          return sp[1]+'.'+sp[2]
        })
        return [...new Set(inputArray.concat(outputArray))][0]
      })
      await this.loadClients(clientsDevices)
    },
    loadClients: async function (filts) {

      const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
        {
          topic: DEFAULT_TOPICS.SERVICES.CLIENT_GET_LIST
        }
      ) : null
      
      const sList = res?.elements ?? {}
      
      const clientFilts = filts.map(val => {
        return val.split('.')[0]
      })

      const deviceFilts = filts.map(val => {
        return val.split('.')[1]
      })


      try {
        this.clientsOfInterest = sList.filter(val => val.state === 2 && clientFilts.includes(val.id))
        this.clientsOfInterest.forEach(client => {
          client.devices = client.devices.filter(val => deviceFilts.includes(val.name))
        })
      } catch {
        this.msg = 'No Clients for this session are available.'
        this.trigger= !this.trigger
      }
      
    },
    registerClientNodes: async function () {
      this.clientsOfInterest.forEach(client => {
        client.devices.forEach(device => {
          this.registerClientNode(client.name, device, device.components)
        })
      })
    },
    calcPostions: async function () {
      console.log('TODO CALC POSITIONS')
    },
    addSessionNodes: async function () {
      this.selectedSession.processingModules.forEach(proc => {
        proc.position = [600,200];
        this.addNode("Sessions/"+this.selectedSession.name+"/"+proc.name, proc.position )
      })
    },
    addClientNodes: async function () {
      this.clientsOfInterest.forEach(client => {
        client.devices.forEach(device => {
          device.position = [200,200];
          this.addNode("Clients/"+client.name+"/"+device.name, device.position)
        })
      })
    },
    showSessionPipeline: async function() {
      await this.loadProcModules()
      await this.registerSessionNodes()
      await this.loadComponents()
      await this.registerClientNodes()
      await this.calcPostions()
      await this.addClientNodes()
      await this.addSessionNodes()
      // console.log(this.selectedSession)
    },
    loadOps: async function({ action/*, callback*/ }) {
      if (action === LOAD_ROOT_OPTIONS) {

        const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
          {
            topic: DEFAULT_TOPICS.SERVICES.SESSION_RUNTIME_GET_LIST
          }
        ) : null
    
        const sList = res?.sessionList ?? {}
        // console.log(sList)
        try { 
          this.availableSessions = sList.filter(val => val.status === 1).map(val => {
              return { 
                id: val.id,
                label: `${val.name}`
              }
            })
        } catch {
          this.msg = 'No sessions available.'
          this.trigger= !this.trigger// console.log('An empty or a invalid response from the server.')
        }
      }
    }
  }
}
</script>

<style scoped>
.header-editor {
  border: 1px solid;
  padding: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>