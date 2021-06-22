<template>
  <div>
  <Error :alert="trigger" :msg="msg"/>
  <div class="header-editor">
    <span style="padding-right: 5px;">Sessions:</span>
    <treeselect v-model="selectedSessionId" 
      @input="showSessionPipeline()"
      :load-options="loadOps" 
      :options="availableSessions" 
      :auto-load-root-options="false" 
      :multiple="false" 
      placeholder="Sessions..."/>
  </div>
  <b-row no-gutters>
    <b-col>
      <b-tabs content-class="mt-3">
        <b-tab title="Clients" active>
          <b-list-group>
            <b-list-group-item>Add Clients to the Graph.</b-list-group-item>
            <b-list-group-item v-for="clientDev in addClientsList" :key="clientDev.id">
              <b-card
                title="Client.Device.Name"
              >
              <b-card-text>{{clientDev.id}}.{{clientDev.name}}</b-card-text>
              <div >
                <b-button @click="addClientToGraph(clientDev)" variant="outline-primary" style="margin: 2px;">Add Client to the Graph</b-button>
                <b-button @click="removeClientNode(clientDev)" variant="outline-primary" style="margin: 2px;">Remove Client from the Graph</b-button>
              </div>
              </b-card>
            </b-list-group-item>
          </b-list-group>
        </b-tab>
        <b-tab title="Processing Modules">
          <b-list-group>
            <b-list-group-item>Add Processing Modules to the Graph.</b-list-group-item>
            <b-list-group-item v-for="proc in addProcsList" :key="proc.id">
              <b-card
                title="Processing Module Name"
              >
              <b-card-text>{{proc.id}}.{{proc.name}}</b-card-text>
              <div >
                <b-button @click="addProcToGraph(proc)" variant="outline-primary" style="margin: 2px;">Add PM the Graph</b-button>
                <b-button @click="removeProcNode(proc)" variant="outline-primary" style="margin: 2px;">Remove PM from the Graph</b-button>
              </div>
              </b-card>
            </b-list-group-item>
          </b-list-group>
        </b-tab>
      </b-tabs>
    </b-col><b-col>
    <canvas id="canvas" class='litegraph' width='1024' height='720' style='border: 1px solid;'></canvas>
    </b-col>
  </b-row>
  </div>
</template>
<script>

import * as litegraph from "litegraph.js";
// import uuid4 from "uuid";
import "litegraph.js/css/litegraph.css";

import { UbiiClientService } from '@tum-far/ubii-node-webbrowser';
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

      clientsOfInterest: null,

      addClientsList: [],
      addProcsList: [],

      ClSeNodes: []
    };
  },

  mounted() {
    UbiiClientService.instance.setName('ubii-node-webbrowser VueJS Test');
    UbiiClientService.instance.setHTTPS(false);
    UbiiClientService.instance.connect(this.serverIP, this.servicePort);
    this.graph = new litegraph.LGraph();
    this.lGraphCanvas = new litegraph.LGraphCanvas('#canvas', this.graph);
    this.graph.start();
    
    litegraph.LiteGraph.clearRegisteredTypes()        
  },
  methods: {
    clearBeforeRender: async function() {
      this.ClSeNodes = []
      this.addClientsList = []
      this.addProcsList = []
      this.graph.clear()
      litegraph.LiteGraph.clearRegisteredTypes()
    },
    ubiiGetResult: async function(topic) {
      await UbiiClientService.instance.waitForConnection()
      const res = await UbiiClientService.instance.client.callService(
        {
          topic: topic
        }
      ) 
      return res

    },
    addClientsToList: async function() {
      
      this.clientsOfInterest.filter(val => val.state === 0).forEach(client => {
        client.devices.forEach(dev => {
          this.addClientsList.push(
            {
              id: dev.clientId+'.'+dev.id,
              name: dev.name
            }
          )
        })
      })
    },
    addProcsToList: async function() {
      const res = await this.ubiiGetResult(DEFAULT_TOPICS.SERVICES.PM_DATABASE_GET_LIST)
      const pList = res?.processingModuleList.elements ?? {}

      this.addProcsList = pList.filter(val => val.sessionId === this.selectedSession.id)
      console.log(this.addProcsList)
    },
    loadSession: async function () {
      const res = await this.ubiiGetResult(DEFAULT_TOPICS.SERVICES.SESSION_RUNTIME_GET_LIST)
      const sList = res?.sessionList.elements ?? {}

      try { 
        this.selectedSession = sList.filter(val => val.status === 1 && this.selectedSessionId == val.id)[0]
      } catch {
        this.msg = 'Selected Session was not found.'
        this.trigger= !this.trigger
      }
    },
    loadClients: async function (filts) {
      const res = await this.ubiiGetResult(DEFAULT_TOPICS.SERVICES.CLIENT_GET_LIST)
      const sList = res?.clientList.elements ?? {}
 
      const clientFilts = filts.map(val => {
        return val.split('.')[0]
      })

      const deviceFilts = filts.map(val => {
        return val.split('.')[1]
      })

      try {
        this.clientsOfInterest = sList.filter(val => val.state === 0 && clientFilts.includes(val.id))
        this.clientsOfInterest.forEach(client => {
          client.devices = client.devices.filter(val => deviceFilts.includes(val.name))
        })
      } catch {
        this.msg = 'No Clients are available for this session.'
        this.trigger= !this.trigger
      }
      
    },
    loadOps: async function({ action/*, callback*/ }) {
      if (action === LOAD_ROOT_OPTIONS) {

        const res = await this.ubiiGetResult(DEFAULT_TOPICS.SERVICES.SESSION_RUNTIME_GET_LIST)
        const sList = res?.sessionList.elements ?? {}

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
    },
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
      node.title_color = "#243";

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
      // this.properties = { precision: 1 };
      node.title = dev.name
      node.title_color = "#345";
      node.slot_start_y = 40;

      node.prototype.onDrawForeground = function(ctx)
      {
        if(this.flags.collapsed)
          return;
        ctx.fillStyle = "#555";
        ctx.fillRect(0,0,this.size[0],40);
        
        ctx.fillStyle = "#AAA";
        ctx.fillText("Client.ID: "+dev.clientId, 2, 15 );
        ctx.fillText("Dev.ID:    "+dev.id, 2, 30 );
      }


      

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
    addNode: function(name, pos, io, type, id) {

      var node_const = litegraph.LiteGraph.createNode(name);
      node_const.pos = pos;
      this.ClSeNodes.push({
            id: id,
            type: type,
            edges: io,
            node: node_const
          })
      this.graph.add(node_const);
    },
    
    registerProcNodesOfSession: async function () {
      if (!this.selectedSession?.processingModules || this.selectedSession.processingModules.length === 0) throw 'Session has no processing modules.'
      this.selectedSession.processingModules.forEach(proc => {
        this.registerProcessNode(this.selectedSession.name, proc, proc.inputs, proc.outputs)
      })
    },
    loadClientsOfSessionAndIO: async function () {
       
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
    addProcNodes: async function () {
      this.selectedSession.processingModules.forEach(proc => {
        const io = this.selectedSession.ioMappings.filter(val => proc.id === val.processingModuleId)[0]
        proc.position = [600,200];
        this.addNode("Sessions/"+this.selectedSession.name+"/"+proc.name, proc.position, io, 'Proc', proc.id )
      })
    },
    addClientNodes: async function () {
      this.clientsOfInterest.forEach(client => {
        client.devices.forEach(device => {
          device.position = [200,200];
          this.addNode("Clients/"+client.name+"/"+device.name, device.position, device.components, 'ClientDevice', device.clientId+'.'+device.id)
        })
      })
    },
    connectNodes: async function () {
      const c = this.ClSeNodes.filter(val => val.type === 'ClientDevice')
      const s = this.ClSeNodes.filter(val => val.type === 'Proc')
      s.forEach(session => {
        session.edges.inputMappings.forEach((i,index) => {
          const dc = c.filter(val => val.edges.filter(e => e.topic === i.topic))[0]
          if (dc !== null) dc.node.connect(index, session.node, index)
        })
        session.edges.outputMappings.forEach((o,index) => {
          const dc = c.filter(val => val.edges.filter(e => e.topic === o.topic))[0]
          if (dc !== null) session.node.connect(index, dc.node, index)
        })
      })
    },
    showSessionPipeline: async function() {
      await this.clearBeforeRender()
      if(!this.selectedSessionId) return
      await this.loadSession()
      try {
        await this.registerProcNodesOfSession()
        await this.loadClientsOfSessionAndIO()
        await this.registerClientNodes()
        await this.addClientsToList();
        await this.addProcsToList();
        await this.calcPostions()
        await this.addClientNodes()
        await this.addProcNodes()
        await this.connectNodes()
      } catch (error) {
        console.log(error)
      }
      
      // console.log(this.selectedSession)
    },
    removeClientNode: async function (c) {
      const cID = c.id.split('.')[0]+'.'+c.id.split('.')[1]
      let nodeIndex = -1
      this.ClSeNodes.forEach((val,index) => {
        if(val.type === 'ClientDevice' && val.id === cID)
          nodeIndex = index
      })
    
      if (nodeIndex >= 0) {
        const cNode = this.ClSeNodes[nodeIndex].node
        this.graph.remove(cNode)
        this.ClSeNodes.splice(nodeIndex,1)
      }
    },
    removeProcNode: async function (p) {
      const pID = p.id.split('.')[0]
      let nodeIndex = -1

      console.log(this.ClSeNodes)

      this.ClSeNodes.forEach((val,index) => {
        if(val.type === 'Proc' && val.id === pID)
          nodeIndex = index
      })

      if (nodeIndex >= 0) {
        const cNode = this.ClSeNodes[nodeIndex].node
        this.graph.remove(cNode)
        this.ClSeNodes.splice(nodeIndex,1)
      }
    },
    addProcToGraph: function (p) {
      const pID = p.id.split('.')[0]
      let nodeIndex = -1

      this.ClSeNodes.forEach((val,index) => {
        if(val.type === 'Proc' && val.id === pID)
          nodeIndex = index
      })

      if (nodeIndex >= 0) {
        const cNode = this.ClSeNodes[nodeIndex].node
        this.lGraphCanvas.selectNode(cNode)
        return
      }
      // Use filter instead of continue
      this.addProcsList.forEach(proc => {
        if (proc.id !== pID) return
        const io = this.selectedSession.ioMappings.filter(val => proc.id === val.processingModuleId)[0]
        proc.position = [600,200];
        this.addNode("Sessions/"+this.selectedSession.name+"/"+proc.name, proc.position, io, 'Proc', proc.id )
      })
    },
    addClientToGraph: function (c) {
      const cID = c.id.split('.')[0]+'.'+c.id.split('.')[1]

      let nodeIndex = -1
      this.ClSeNodes.forEach((val,index) => {
        if(val.type === 'ClientDevice' && val.id === cID)
          nodeIndex = index
      })

      if (nodeIndex >= 0) {
        const cNode = this.ClSeNodes[nodeIndex].node
        this.lGraphCanvas.selectNode(cNode)
        return
      }
      // Use filter instead of continue
      this.clientsOfInterest.forEach(client => {
        if (client.id !== c.id.split('.')[0]) return
        
        client.devices.forEach(device => {
          if (device.id !== c.id.split('.')[1]) return
          
          device.position = [200,200];
          this.addNode("Clients/"+client.name+"/"+device.name, device.position, device.components, 'ClientDevice', device.clientId+'.'+device.id)
        })
      })

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