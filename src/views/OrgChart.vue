<template>
  <div>
    <!-- <label> Enjoei organograma colaborativo</label> -->
    <organization-chart 
      :datasource="ds"
      :pan="true"
    >
    <template slot-scope="{ nodeData }">
      <v-row class="ma-0 pa-0">
        <v-col cols="3" class="pa-0">
          <v-row class="ma-0 pa-0">  
            <v-avatar size="48">
              <v-img :src="nodeData.photo || null"/>          
            </v-avatar>  
          </v-row>
        </v-col>
        <v-col cols="8" class="pa-0">
          <v-row class="ma-0 pa-0 justify-center" >    
            <b><label>{{nodeData.name}}</label></b>      
          </v-row>
          <v-row class="ma-0 pa-0 justify-center">    
            <label style="font-size: 10px">{{nodeData.role}}</label>      
          </v-row>
        </v-col>
        <v-col cols="1" class="pa-0 justify-center">
          <v-row class="ma-0 pa-0">
            <v-icon @click.stop="deletePerson(nodeData)" size="14">mdi-delete</v-icon>
          </v-row>
        </v-col>
      </v-row>
    </template>
    </organization-chart>
    <v-row style="justify-content: center"> 
      <v-form v-model="formModel">
        <v-card width="400">
          <v-card-title>
            Adicionar Pessoa
          </v-card-title>

          <v-card-text>
            <v-text-field
              label="Nome"
              v-model="nodeParams.name"
              :rules="[rules.required, rules.maxCount]"
              counter="15"
            />
            <v-text-field
              label="Foto (Url)"
              v-model="nodeParams.photo"
            />
            <v-text-field
              label="Cargo"
              v-model="nodeParams.role"
              :rules="[rules.required, rules.maxCount]"
              counter="15"
            />
            <v-autocomplete
              v-model="nodeParams.parent_id"
              :items="items"
              auto-select-first
              item-text="name"
              item-value="id"
              label="Chefe"
          ></v-autocomplete>
          </v-card-text>
          <v-card-actions>
            <v-btn @click="addPerson()" :disabled="!formModel">
              Salvar
            </v-btn>
            <v-btn @click="nodeParams = Object.assign({}, defaulNodeParams)">
              Resetar
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-form>      
    </v-row>
  </div>
</template>

<script>
  import OrganizationChart from 'vue-organization-chart'
  import 'vue-organization-chart/dist/orgchart.css'
  import _ from 'lodash'
  import { Socket, Presence } from '@/assets/phoenix.js'

  export default {
    name: 'OrgChart',
    components: {
      OrganizationChart
    },
    data () {
      return {
        socket: null,
        channel: null,
        presence: null,
        isUserOnline: false,
        nodeParams: {},
        items: [],
        dataSet: {
          id: '',
          name: '',
          parent_id: null,
          photo: '',
          role: '',
          children: []
        },
        people: [],
        rules: {
          required: value => !!value || 'Required.',
          maxCount: v => !!v && v.length <= 15 || 'Max 15 characters.'
        },
        formModel: false
      }
    },
    mounted() {
      this.nodeParams = Object.assign({}, this.defaulNodeParams)
      this.enterWebsocket()
    },
    methods: {
      enterWebsocket() {
        this.connectSocket()
        this.connectChannel()
      },
      connectSocket () {
        this.socket = new Socket(`${process.env.WEBSOCKET_URL}/socket`, {params: {username: this.username}})
        this.socket.onError((error) => {
          console.log(`there was an error with the connection! ${error}`)
          this.isUserOnline = false
        })
        this.socket.onClose((error) => {
          console.log(`the connection dropped ${error}`)
          this.isUserOnline = false
        })
        this.socket.connect()
      },
      connectChannel() {
        this.channel = this.socket.channel('room:lobby', {})
        this.presence = new Presence(this.channel)
        this.presence.onSync(() => {
          console.log('presence.list():', this.presence.list())
        })

        this.channel.join()
          .receive('ok', ({data}) => {
            this.isUserOnline = true
            this.people = data
            this.buildDs()
          })
          .receive('error', ({reason}) => console.log('failed join', reason))
          .receive('timeout', () => console.log('Networking issue. Still waiting...'))

        this.channel.on('new_person', (person) => {
          this.people.push(person)
          this.buildDs()
        })

        this.channel.on('delete_person', (person) => {
          var peopleIds = [person.id]
          do{
            _.remove(this.people,(p) => peopleIds.includes(p.id))
            peopleIds = _.reduce(this.people, (newArray, p) => {
              if (_.includes(peopleIds, p.parent_id)) {
                newArray.push(p.id)
              }
              return newArray
            },[])
          } while (peopleIds && peopleIds.length != 0)
          this.people = Object.assign([], this.people)
          this.buildDs()
        })
      },
      findParentNodeAndExecute(parent, node, func) {
        if(parent.id == node.parent_id) {
          func(parent, node)
        } else {
          _.forEach(parent.children, (newParent) => {
            this.findParentNodeAndExecute(newParent, node, func)
          })
        }
      },
      addNode(node) {
        this.findParentNodeAndExecute(this.dataSet, node, (parent, newNode) => {
          if(parent.children == undefined){
            parent.children = new Array()
          } 
          let cp = Object.assign({}, newNode)
          parent.children.push(cp)
        })
      },
      defaulNodeParams() {
        return {
          id: null,
          parent_id: null,
          name: '',
          photo: null,
          role: '',
          children: []       
        }
      },
      defaultDataSet() {
        return {
          id: null,
          name: "Enjoei",
          parent_id: null,
          photo: 'https://media-exp1.licdn.com/dms/image/C4E0BAQHg-JH19qoSpg/company-logo_200_200/0?e=1604534400&v=beta&t=f-3vphkp9P-I-4XwxOI1fFc-NNO8CQQLXWP4xITuSo4',
          role: "Engenharia",
          children: []
        }
      },
      buildDs() {
        this.dataSet = Object.assign({}, this.defaultDataSet()) 
        _.forEach(this.people, (newNode) => {
          this.addNode(newNode)
        })
      },
      addPerson(){
        this.channel.push('new_person', this.nodeParams)
          .receive("error", () => console.log("error to add new person"))
      },
      deletePerson(person){
        this.channel.push('delete_person', person)
          .receive("error", () => console.log("error to delete person"))
      }
    },

    computed: {
      ds() {
        return this.dataSet
      }
    },
    watch:{
      people: function (val) {
        this.items = val.slice()
        this.items.push({id: null, name: 'Enjoei'})
      }
    }
  }
</script>

<style >
.orgchart {
  background: none!important;
}
.node {
  width: 200px!important;
  background: #ededed;
  border-radius: 10px;
  margin: 0px 10px 0px 10px !important;
}

.orgchart-container {
  height: 60vh;
}

</style>