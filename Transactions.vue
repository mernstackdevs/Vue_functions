<template>
    <v-layout style="font-family: 'Poppins', sans-serif;height:100%">
        <AppBar @toggle-drawer="$refs.drawer.drawer = !$refs.drawer.drawer" />
        <Navigation ref="drawer" />
        <v-container fluid>
            <!-- filters -->
                <v-row>
                    <v-col cols=12>
                        <h1 class="display-2 py-5">Transactions</h1>
                        <v-card>
                            <v-card-title class="grey darken-4 py-3">
                                <h4>Filters</h4>
                                <v-col class="d-flex justify-end" style="padding:0px">
                                    <v-btn rounded v-if="is_filter" class="blue lighten-2 mr-3" @click=" is_filter=false,page=1,fromDateVal='',toDateVal='',min_amount='',max_amount='',from_user='',to_user='',paginate()">Reset</v-btn>
                                    <v-btn rounded class="red lighten-1" @click="filter_txns">Submit</v-btn>
                                </v-col>
                            </v-card-title>
                            <v-card-text class="pb-0 mb-3">
                                <v-row class="mt-1">
                                    <v-col cols=2>
                                        <v-select
                                        label="From user"
                                        v-model="from_user"
                                        :items="all_users"
                                        item-text="name"
                                        item-value="pk"
                                        dense autocomplete outlined
                                        >
                                        </v-select>
                                    </v-col>
                                     <v-col cols=2>
                                        <v-select
                                        label="To user"
                                        v-model="to_user"
                                        :items="all_users"
                                        item-text="name"
                                        item-value="pk"
                                        dense autocomplete outlined
                                        >
                                        </v-select>
                                    </v-col>
                                    <v-col cols=2>
                                        <v-menu
                                            v-model="fromDateMenu"
                                            :close-on-content-click="false"
                                            transition="scale-transition"
                                            offset-y
                                            >
                                            <template v-slot:activator="{ on }">
                                                <v-text-field
                                                label="From date"
                                                append-icon="mdi-calendar"
                                                readonly
                                                :value="fromDateDisp"
                                                v-on="on" outlined dense
                                                ></v-text-field>
                                            </template>
                                            <v-date-picker
                                                v-model="fromDateVal"
                                                no-title
                                                min="2021-01-01"
                                            ></v-date-picker>
                                            </v-menu>
                                    </v-col>
                                    <v-col cols=3>
                                        <v-menu
                                            v-model="toDateMenu"
                                            :close-on-content-click="false"
                                            transition="scale-transition"
                                            offset-y
                                            >
                                            <template v-slot:activator="{ on }">
                                                <v-text-field
                                                label="To date"
                                                append-icon="mdi-calendar"
                                                readonly
                                                :value="toDateDisp"
                                                v-on="on" outlined dense
                                                ></v-text-field>
                                            </template>
                                            <v-date-picker
                                                v-model="toDateVal"
                                                no-title
                                                min="2021-01-01"
                                            ></v-date-picker>
                                            </v-menu>
                                    </v-col>
                                    <v-form v-model="amount_val" lazy-validation ref="amountForm" style="display:contents">
                                    <v-col cols=3 class="d-flex d-inline">
                                        <v-text-field
                                        label="Min. amount" @keydown.space.prevent :rules="amountRules"
                                        v-model="min_amount" outlined dense class="mr-3" 
                                        ></v-text-field>
                                        <v-text-field
                                        label="Max. amount" @keydown.space.prevent :rules="amountRules"
                                        v-model="max_amount" outlined dense
                                        ></v-text-field>
                                    </v-col>
                                    </v-form>                                    
                                </v-row>
                            </v-card-text>
                        </v-card>
                    </v-col>
                </v-row>
            <!-- filters -->

            <!-- table data -->
            <v-row>
                <v-col cols=12>
                    <v-data-table
                    :headers='headers'
                    :items='transactions'
                    item-key="pk"
                    hide-default-footer
                    :loading="loading"
                    :items-per-page="perPage"
                    >
                        <template v-slot:item.from_account_td="{item}">
                            <v-avatar size=35>
                                <v-img :src="backend_url+item.from_account_img"></v-img>
                            </v-avatar>
                            <span class="pl-3">{{item.from_account}}</span>
                        </template>
                        <template v-slot:item.to_td="{item}">
                            <v-avatar size=35>
                                <v-img :src="backend_url+item.to_img"></v-img>
                            </v-avatar>
                            <span class="pl-3">{{item.to}}</span>
                        </template>
                    </v-data-table>
                    <v-pagination
                        class="py-2 grey darken-4"
                        v-model="page"
                        :length="pages"
                        @input="paginate"
                        color="red accent-5"
                        :total-visible="10"
                    ></v-pagination>
                </v-col>
            </v-row>
            <!-- table data -->
            
        </v-container>
    </v-layout>
</template>

<script>
import Navigation from '../components/Navigation.vue'
import AppBar from '../components/AppBar.vue'
import axios from 'axios'
import router from '../router'
import { mapActions} from "vuex";
var regex = /^$|(?!0*[.,]0*$|[.,]0*$|0*$)\d+[,.]?\d{0,2}$/
export default {
    name:'Transactions',
    components:{
        'Navigation':Navigation,
        'AppBar':AppBar
    },
    data: function(){
        return {
            transactions:[],
            all_users:[],
            page:1,
            pages:0,
            loading:true,
            perPage:10,
            backend_url : this.$store.state.backend_url,
            from_user:'',
            to_user:'',
            max_amount:'',
            min_amount:'',
            amount_val:false,
            fromDateMenu: false,
            fromDateVal: '',
            toDateMenu: false,
            toDateVal: '',
            is_filter:false,
            tz:'',
            headers:[
                {text:'User', value:'user', align:'start', sortable:false},
                {text:'Source', value:'from_account_td', align:'start', sortable:false},
                {text:'To User', value:'recv_user', align:'start', sortable:false},
                {text:'Destination', value:'to_td', align:'start', sortable:false},
                {text:'Amount', value:'amount', align:'start', sortable:false},
                {text:'Status', value:'status', align:'start', sortable:false},
                {text:'Date/Time', value:'created_at', sortable:false},
                {text:'Admin Commission', value:'fees', sortable:false, width:'10px',align:'center',},
            ],
            amountRules:[
                v =>  regex.test(v) || 'Value should be numeric & greater than 0.' ,
            ]
        }
    },
    methods:{
        ...mapActions(["snackBar", "overLay"]),
        paginate(){
            this.loading = true
            var header = {
                "Authorization":localStorage.getItem('token')
            }
            var url = process.env.VUE_APP_BACKEND_URL+"/adminAPI/all_transactions?page="+this.page+"&from_date="+this.fromDateVal+'&to_date='+this.toDateVal+'&min_amount='+this.min_amount+'&max_amount='+this.max_amount+'&timezone='+this.tz+'&from_user='+this.from_user+'&to_user='+this.to_user
            axios.get(url, {headers: header}).then(resp => {
                this.transactions = resp.data.data
                this.pages = resp.data.pages
                this.perPage = resp.data.page_count
                this.loading = false
            }).catch(error => {
                if(error.response.status==401){
                    localStorage.removeItem('user_login');
                    localStorage.removeItem('token');
                    router.push('/')
                }
                this.loading = false
                this.snackBar(error.response.data.message)
            })
        },
        filter_txns(){
            if(!this.$refs.amountForm.validate()){
                return false
            }
            if(this.from_user||this.fromDateVal||this.toDateVal||this.min_amount||this.max_amount||this.to_user){
                this.is_filter = true
                this.page=1
                this.loading = true
                var header = {
                    "Authorization":localStorage.getItem('token')
                }
                var url = process.env.VUE_APP_BACKEND_URL+"/adminAPI/all_transactions?from_date="+this.fromDateVal+'&to_date='+this.toDateVal+'&min_amount='+this.min_amount+'&max_amount='+this.max_amount+'&from_user='+this.from_user+'&to_user='+this.to_user+'&timezone='+this.tz
                axios.get(url, {headers: header}).then(resp => {
                    this.transactions = resp.data.data
                    this.pages = resp.data.pages
                    this.perPage = resp.data.page_count
                    this.loading = false
                }).catch(error => {
                    if(error.response.status==401){
                        localStorage.removeItem('user_login');
                        localStorage.removeItem('token');
                        router.push('/')
                    }
                    this.loading = false
                    this.snackBar(error.response.data.message)
                })
            }
        },
        reset_filters(){
            this.is_filter = false
            this.page=1
            this.fromDateVal=''
            this.toDateVal=''
            this.min_amount=''
            this.max_amount=''
            this.from_user=''
            this.to_user=''
            this.paginate()
        }
    },
    mounted(){
        this.overLay(true)
        this.tz = Intl.DateTimeFormat().resolvedOptions().timeZone
        var header = {
            "Authorization":localStorage.getItem('token')
        }
        var url = process.env.VUE_APP_BACKEND_URL+"/adminAPI/all_transactions?timezone="+this.tz
        axios.get(url, {headers: header}).then(resp => {
            this.transactions = resp.data.data
            this.all_users = resp.data.users
            this.pages = resp.data.pages
            this.perPage = resp.data.page_count
            this.loading = false
            this.overLay(false)
        }).catch(error => {
            if(error.response.status==401){
                localStorage.removeItem('user_login');
                localStorage.removeItem('token');
                this.snackBar(error.response.data.message)
                router.push('/')
                this.overLay(false)
                return;
            }
            this.snackBar(error.response.data.message)
            this.overLay(false)
            router.go(-1)
        })
    },
  computed:{
      fromDateDisp() {
          if(this.fromDateVal){
            var datePart = this.fromDateVal.match(/\d+/g)
            var year = datePart[0].substring(2)
            var month = datePart[1]
            var day = datePart[2]
            return month+'-'+day+'-'+year
          }
          return this.fromDateVal
      },
      toDateDisp() {
        if(this.toDateVal){
            var datePart = this.toDateVal.match(/\d+/g)
            var year = datePart[0].substring(2)
            var month = datePart[1]
            var day = datePart[2]
            return month+'-'+day+'-'+year
          }
          return this.toDateVal
      },
  },
  created(){
        document.title = 'Transactions - PayRink Admin'
    }
}
</script>

<style scoped lang="scss">
::v-deep .v-label{
    font-size: 15px !important;
}

</style>