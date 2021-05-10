<template>
  <div style="height: 4086px;">
    <ExampleMousePointer/>
    <Error :alert="trigger" :msg="msg"/>
    <div class="server-connection">
      <label>IP</label>
      <input class="input" v-model="serverIP" />
      <label>Service Port</label>
      <input class="input" v-model="servicePort" />
      <button class="input btn-connect" @click="connectUbii()">⟲</button>
    </div>
    <div style="display: flex; justify-content: center">
      <b-button variant="outline-primary" @click="getServiceList()">ShowAllAvailableServices</b-button>
      <b-button variant="outline-primary" @click="getClients()">ShowAllConnectedClients</b-button>
      <!-- <b-button variant="outline-primary" @click="getSessionList()">ShowAllSessions</b-button> -->
    </div>
    <div style="display: flex; justify-content: center">
      <treeselect v-model="clients" :load-options="loadClients" :options="clientOptions" :auto-load-root-options="false" :multiple="true" placeholder="Clients..." style="width: 500px;"/>
      <treeselect v-model="sessions" :load-options="loadSession" :options="sessionOptions" :auto-load-root-options="false" :multiple="true" placeholder="Sessions..." style="width: 500px;"/>
      <!-- <div class="col">
        <ul id="example-2">
          <li v-for="item in connected_clients" :key="item.id">
            {{ item.id }}: {{ item.name }}
          </li>
        </ul>
      </div>
      <div class="col">
        <ul id="example-3">
          <li v-for="item in connected_clients" :key="item.id">
            {{ item.id }}: {{ item.name }}
          </li>
        </ul>
      </div> -->
    </div>
    <div>
      <ul id="example-1">
        <li v-for="(item, key) in obj_services" :key="key">
          {{ key }}: {{ item }}
        </li>
      </ul>
    </div>
  </div>

</template>

<script>
import UbiiClientService from '../ubiiNode/ubiiClientService'
import { DEFAULT_TOPICS } from '@tum-far/ubii-msg-formats';

import ExampleMousePointer from '../components/MouseExample.vue'
import Error from '../components/Error.vue'

// import the component
import Treeselect from '@riophae/vue-treeselect'
import { LOAD_ROOT_OPTIONS } from '@riophae/vue-treeselect'

// import the styles
import '@riophae/vue-treeselect/dist/vue-treeselect.css'

export default {
  name: 'PageIndex',
  components: {
    ExampleMousePointer,
    Treeselect,
    Error
  },
  data: function() {
    return {
      serverIP: window.location.hostname,
      servicePort: 8102,
      obj_services: null,
      connected_clients: null,
      trigger: 0,
      msg: '',

      sessions: null,
      sessionOptions: null,

      clients: null,
      clientOptions: null
    };
  },
  methods: {
    connectUbii: function() {
      UbiiClientService.setName('ubii-node-webbrowser VueJS Test');
      UbiiClientService.connect(this.serverIP, this.servicePort);
    },
    getClients: async function() {
      const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
        {
          topic: DEFAULT_TOPICS.SERVICES.CLIENT_GET_LIST
        }
      ) : null
      
      //console.log(res)
      // const constJson = res?.server?.constantsJson ?? '{}'
  
      try { 
        this.connected_clients = res.elements
      } catch {
        this.msg = 'An empty or a invalid response from the server.'
        this.trigger= !this.trigger// console.log('An empty or a invalid response from the server.')
      }
    },
    getServiceList: async function() {
      const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
        {
          topic: DEFAULT_TOPICS.SERVICES.SERVER_CONFIG
        }
      ) : null

      const constJson = res?.server?.constantsJson ?? '{}'

      try { 
        const jRes = await JSON.parse(constJson)
        this.obj_services = jRes['DEFAULT_TOPICS']['SERVICES']
      } catch {
        // console.log('An empty or a invalid response from the server.')
      }
    },
    loadClients: async function ({action}) {
      if (action === LOAD_ROOT_OPTIONS) {
        const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
          {
            topic: DEFAULT_TOPICS.SERVICES.CLIENT_GET_LIST
          }
        ) : null
        
        const sList = res?.elements ?? {}
        console.log(sList)

        try {
          this.clientOptions = sList.filter(val => val.state === 0).map(val => {
            return { 
                id: val.id,
                label: `${val.name}`,
                children: val.devices.map(child => {
                  return { 
                    id: `${val.id}.${child.id}`,
                    label: `${child.name}`,
                    children: child.components.map(comp => {
                      return {
                        id: `${val.id}.${child.id}.${comp.id}`,
                        label: `${comp.topic}`
                      }
                    })
                  }
                })
            }
          })
        } catch {
          console.log('error')
        }
      }
    },
    loadSession: async function({ action/*, callback*/ }) {
      if (action === LOAD_ROOT_OPTIONS) {

        const res = (UbiiClientService.isConnected()) ? await UbiiClientService.client.callService(
          {
            topic: DEFAULT_TOPICS.SERVICES.SESSION_RUNTIME_GET_LIST
          }
        ) : null
    
        const sList = res?.sessionList ?? {}

        try { 
          this.sessionOptions = sList.filter(val => val.status === 1).map(val => {
              return { 
                id: val.id,
                label: `${val.name}`,
                children: val.processingModules.map(child => {
                  return { 
                    id: `${val.id}.${child.id}`,
                    label: `${child.name}`,
                    children: [
                        {
                          id: `${val.id}.${child.id}.inputs`,
                          label: 'inputs',
                          children: child.inputs.map((inp) => {
                            return { 
                              id: `${val.id}.${child.id}.inputs.${inp.internalName}`,
                              label: `${inp.internalName}`

                            }
                          })
                        },
                        {
                          id: `${val.id}.${child.id}.outputs`,
                          label: 'outputs',
                          children: child.outputs.map((out) => {
                            return {
                              id: `${val.id}.${child.id}.outputs.${out.internalName}`,
                              label: `${out.internalName}`
                            }
                          })
                        }
                      ]
                  }
                })
              }
            })
        } catch {
          console.log('An Error occured.')
        }
      }
    },

  }
}
</script>
