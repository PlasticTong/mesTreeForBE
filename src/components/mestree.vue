<template>
    <div>
        <div class="mar-title">
            <h2 style="margin: 5px 5px;font-size: 20px;">消息流</h2>
            <el-button @click="importall"
                style="border-radius: 10px; margin-left: 10px; background-color: aquamarine; color: blue;">导入</el-button>
            <el-button @click="exportall"
                style="border-radius: 10px; margin-left: 10px; background-color: aquamarine; color: rgb(221, 103, 24);">导出</el-button>
            <el-dialog :visible.sync="dialogVisble" width="500px" :before-close="handleClose">
                <div>时间片：
                    <el-input v-model="timeSlice" placeholder="请输入" style="width: 20%;">
                    </el-input>
                    *10分钟
                </div>

                <!-- <div style="display: flex;flex-direction: row;">时间片:<el-input v-model="formInline.threshold" placeholder="时间片" style="width: 20%;"></el-input></div> -->
                <el-upload action="#" :http-request="uploadFileTest" :limit="1" :drag=true style="margin-top: 10px;">
                    <i class="el-icon-upload"></i>
                    <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
                    <!-- <el-button
                        style="border-radius: 10px; margin-left: 10px; background-color: aquamarine; color: blue;">导入文件</el-button> -->
                </el-upload>
                <!-- <h2 style="margin: 5px 5px;font-size: 10px;margin-left: 10px;margin-top: 12px;">当前数据：{{ this.filename }}
                </h2> -->
            </el-dialog>


        </div>
        <div>
            <el-tabs v-model="activeName">
                <el-tab-pane label="消息流" name="first" style="display: flex;flex-direction: column;">
                    <div style="display: flex;flex-direction: row;">
                        <el-input v-model="accY" placeholder="纵轴精确度" style="width: 10%;margin-right: 10px;"></el-input>
                        <el-button @click="generateVis2"
                            style="border-radius: 10px;  background-color: aquamarine; color: blue;">绘制</el-button>
                    </div>
                    <svg id="chart" width="800" height="500"></svg>
                </el-tab-pane>
                <el-tab-pane label="网络数据" name="second">
                    <el-form :inline="true" :model="formInline" class="demo-form-inline">
                        <el-form-item label="IP段">
                            <el-input v-model="formInline.user" placeholder="IP段"></el-input>
                        </el-form-item>
                        <el-form-item label="时间">
                            <el-col :span="11"><el-input type="datetime-local" v-model="formInline.datastart"
                                    placeholder="起始时间"></el-input></el-col>
                            <el-col class="line" :span="2">-</el-col>
                            <el-col :span="11"><el-input type="datetime-local" v-model="formInline.dataend"
                                    placeholder="终止时间"></el-input></el-col>
                        </el-form-item>
                        <el-form-item label="跳数">
                            <el-input v-model="formInline.hop" placeholder="跳数"></el-input>
                        </el-form-item>
                        <el-form-item label="时间阈值">
                            <div>
                                <el-input v-model="formInline.threshold" placeholder="时间阈值"
                                    style="width: 70%;"></el-input>*{{ this.timeSlice * 10 }}分钟
                            </div>
                            <!-- <el-col :span="6"><el-select v-model="value" placeholder="请选择">
                                    <el-option v-for="item in thresholdOptions" :key="item.value" :label="item.label"
                                        :value="item.value">
                                    </el-option>

                                </el-select>
                            </el-col> -->
                        </el-form-item>
                        <el-form-item>
                            <el-button type="primary" @click="onSubmit">查询</el-button>
                        </el-form-item>
                    </el-form>
                    <el-table :data="tableDataForShow">
                        <!-- <el-table-column prop="id" label="ID" width="55" align="center"></el-table-column> -->
                        <el-table-column prop="source" label="起始点"></el-table-column>
                        <el-table-column prop="target" label="目标点">
                        </el-table-column>
                        <el-table-column prop="time" label="传递时间"></el-table-column>
                        <!-- <el-table-column prop="content" label="内容"></el-table-column> -->
                    </el-table>
                    <!-- <div class="pagination"> -->
                    <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange"
                        :current-page=page.index :page-sizes="[5, 10, 15]" :page-size=page.size
                        layout="total, sizes, prev, pager, next, jumper" :total=page.total>
                    </el-pagination>
                    <!-- </div> -->
                </el-tab-pane>
                <!-- <el-tab-pane label="节点信息" name="third">

                </el-tab-pane> -->
            </el-tabs>
        </div>
    </div>
</template>
<script>
// import mestable from "./mesinfo.vue";
// import usertable from "./userinfo.vue";
import * as d3 from 'd3';
// import { ElDrawer } from 'element-plus'
// import Edata from '../../public/tablemes.json'
// import Vdata from '../../public/table.json'
// import Mdata from '../../public/Markov.json'
// import store from "../store/mesinfo"
// import { ElMessage, ElMessageBox } from "element-plus";
// import { Delete, Edit, Search, Plus, Pointer } from "@element-plus/icons-vue";
// import FileSaver from 'file-saver'
// import { objectPick } from '@vueuse/shared';
// import { linksUserCho } from "../api/index.ts"
// import { fetchMesData, testflask, mutiDraw, mutiCross, markovData } from "../api/index";
import ElementUI from 'element-ui';
import Vue from 'vue'
import 'element-ui/lib/theme-chalk/index.css';
import axios from "axios"

Vue.use(ElementUI);

export default {
    name: 'mestree',
    data() {
        return {
            selectvalue: [0, 10],
            minn: undefined,
            maxx: undefined,
            k: 4,
            Mlist: [],
            currentRow: undefined,
            marColor: "black",
            selectColor: "green",
            oldCurrentRow: undefined,
            messageColor: "#DCDCDC",
            threshold: 6, //阈值s
            drawer: false,
            direction: 'rtl',
            activeName: "first",
            tableDataForShow: null,
            tableDataAll: null,
            currentPage4: 4,
            page: {
                size: 10,
                index: 1,
                total: 0
            },
            formInline: {
                user: '',//IP
                hop: '',//跳数
                datastart: '',//起始时间
                dataend: '',//终止时间
                threshold: 1,//阈值
                timestart: '',//换算成毫秒的时间
                timeend: '',//换算成毫秒的时间
                thresholdByTime: '',//换算成毫秒的阈值
            },
            fileshow: false,
            filename: "无",
            dialogVisble: false,
            tabledata: null,
            tabledataSearch: null,

            // options: [{
            //     value: 60000,
            //     label: '分钟'
            // }, {
            //     value: 3600000,
            //     label: '小时'
            // }, {
            //     value: 86400000,
            //     label: '天'
            // }],
            value: '',


            thresholdOptions: [{
                value: 60000,
                label: '分钟'
            }, {
                value: 3600000,
                label: '小时'
            }, {
                value: 86400000,
                label: '天'
            }],

            accY: "",
            filterresFromUser: [],

            timeSlice: ""

        };
    },
    computed: {
        draw() {
            return [this.tabledata, this.tabledataSearch]
        }

    },
    // watch: {
    //     draw: {
    //         deep: true,
    //         handler() {
    //             this.generateVis2()

    //         }
    //     }

    // },
    methods: {
        importall() {
            this.dialogVisble = true
        },
        handleClose() {
            this.dialogVisble = false
        },
        exportall() {
            console.log(this.filterresFromUser);
        },
        async uploadFileTest(item) {
            this.tabledataSearch = null
            let formdataw = new FormData()
            formdataw.append('file', item.file)
            await axios({
                url: 'http://127.0.0.1:5002/mesBE/upload',
                method: "POST",
                data: formdataw
            }).then(res => {
                this.filename = res.data
            })

            await axios({
                url: 'http://127.0.0.1:5002/mesBE/read',
                method: "POST",
                data: { name: this.filename }
            }).then(res => {
                this.tabledata = res.data
            })
            for (let i = 0; i < this.tabledata.length; i++) {
                this.tabledata[i].timesecond = Date.parse(this.tabledata[i].time)
            }
            this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            this.page.total = this.tabledata.length


        },
        async onSubmit() {
            this.formInline.timestart = Date.parse(this.formInline.datastart)
            this.formInline.timeend = Date.parse(this.formInline.dataend)
            this.formInline.thresholdByTime = this.formInline.threshold * this.value
            // console.log((Date.parse(this.formInline.dataend) - Date.parse(this.formInline.datastart)) / (1000 * 60 * 60 * 24));
            await axios({
                url: 'http://127.0.0.1:5002/mesBE/searchData',
                method: "POST",
                data: {
                    data: this.tabledata,
                    form: this.formInline
                }
            }).then(res => {
                this.tabledataSearch = res.data
            })
            this.tableDataForShow = this.tabledataSearch.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            this.page.total = this.tabledataSearch.length
            console.log(this.tabledataSearch);
            d3.select("#maingroup").remove()

        },
        handleSizeChange(val) {
            console.log(`每页 ${val} 条`);
            this.page.size = val
            if (this.tabledataSearch != null) {
                this.tableDataForShow = this.tabledataSearch.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            } else {
                this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            }

        },
        handleCurrentChange(val) {
            console.log(`当前页: ${val}`);
            this.page.index = val
            if (this.tabledataSearch != null) {
                this.tableDataForShow = this.tabledataSearch.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            } else {
                this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            }
        },
        generateVis2() {
            if (this.accY == '') {
                this.accY = 1
            }
            let that = this;
            let IPfromChoose = []
            if (this.formInline.user != "") {
                IPfromChoose = this.formInline.user.split(";")
                console.log(IPfromChoose);
            }

            function isIPInNetwork(ip, network) {
                var ipParts = ip.split(".");
                var networkParts = network.split(".");

                // 判断 IP 地址是否在网络地址中
                for (var i = 0; i < 3; i++) {
                    if (ipParts[i] !== networkParts[i]) {
                        return false;
                    }
                }

                return true;
            }
            //计算最大值最小值
            // let filterMesData = [];
            // filterMesData = {
            //     "list": [
            //         {
            //             "id": 1,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.3",
            //             "time": 1,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 2,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.4",
            //             "time": 1,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 3,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.6",
            //             "time": 1,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 4,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.7",
            //             "time": 1,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 5,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.9",
            //             "time": 1,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 6,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.2",
            //             "time": 1,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 7,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.5",
            //             "time": 1,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 8,
            //             "source": "192.168.1.5",
            //             "target": "192.168.1.9",
            //             "time": 1,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 9,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.10",
            //             "time": 1,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 10,
            //             "source": "192.168.1.8",
            //             "target": "192.168.1.11",
            //             "time": 1,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 11,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.3",
            //             "time": 2,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 12,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.4",
            //             "time": 2,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 13,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.6",
            //             "time": 2,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 14,
            //             "source": "192.168.1.5",
            //             "target": "192.168.1.9",
            //             "time": 2,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 15,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.10",
            //             "time": 2,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 16,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.11",
            //             "time": 2,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 17,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.8",
            //             "time": 2,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 18,
            //             "source": "192.168.1.8",
            //             "target": "192.168.1.11",
            //             "time": 2,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 19,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.10",
            //             "time": 2,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 20,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.11",
            //             "time": 2,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 21,
            //             "source": "192.168.1.5",
            //             "target": "192.168.1.9",
            //             "time": 3,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 22,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.10",
            //             "time": 3,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 23,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.11",
            //             "time": 3,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 24,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.3",
            //             "time": 3,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 25,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.4",
            //             "time": 3,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 26,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.6",
            //             "time": 3,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 27,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.11",
            //             "time": 3,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 28,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.12",
            //             "time": 3,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 29,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.13",
            //             "time": 3,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 30,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.14",
            //             "time": 3,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 31,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.15",
            //             "time": 4,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 32,
            //             "source": "192.168.1.2",
            //             "target": "192.168.1.16",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 33,
            //             "source": "192.168.1.5",
            //             "target": "192.168.1.9",
            //             "time": 4,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 34,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.10",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 35,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.11",
            //             "time": 4,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 36,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.3",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 37,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.4",
            //             "time": 4,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 38,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.6",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 39,
            //             "source": "192.168.1.4",
            //             "target": "192.168.1.8",
            //             "time": 4,
            //             "content": "下班"
            //         },
            //         {
            //             "id": 40,
            //             "source": "192.168.1.8",
            //             "target": "192.168.1.11",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 41,
            //             "source": "192.168.1.111",
            //             "target": "192.168.1.112",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 42,
            //             "source": "192.168.1.112",
            //             "target": "192.168.1.113",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 43,
            //             "source": "192.168.1.113",
            //             "target": "192.168.1.114",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 44,
            //             "source": "192.168.1.115",
            //             "target": "192.168.1.116",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 45,
            //             "source": "192.168.1.116",
            //             "target": "192.168.1.117",
            //             "time": 4,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 46,
            //             "source": "192.168.1.111",
            //             "target": "192.168.1.112",
            //             "time": 14,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 47,
            //             "source": "192.168.1.112",
            //             "target": "192.168.1.113",
            //             "time": 14,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 48,
            //             "source": "192.168.1.114",
            //             "target": "192.168.1.115",
            //             "time": 14,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 49,
            //             "source": "192.168.1.115",
            //             "target": "192.168.1.116",
            //             "time": 14,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 50,
            //             "source": "192.168.1.116",
            //             "target": "192.168.1.117",
            //             "time": 14,
            //             "content": "学习"
            //         },
            //         {
            //             "id": 51,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.13",
            //             "time": 14,
            //             "content": "下班"
            //         }
            //     ],
            //     "pageTotal": 51
            // }
            // this.tableDataForShow = filterMesData.list.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            // this.page.total = filterMesData.list.length
            // var maxTime = 0;
            // var minTime = 0;
            // if (filterMesData != []) {
            //     maxTime = filterMesData.list.reduce(function (prev, curr) {
            //         return prev.time > curr.time ? prev : curr;
            //     }, {});
            //     minTime = filterMesData.list.reduce(function (prev, curr) {
            //         return prev.time < curr.time ? prev : curr;
            //     }, {});
            // }
            // this.maxx = maxTime
            // this.minn = minTime

            let filterMesDataByHold = [];
            if (this.tabledataSearch == null) {
                filterMesDataByHold = this.tabledata
            } else {
                filterMesDataByHold = this.tabledataSearch
            }
            // filterMesDataByHold = this.tabledata

            let filterUserData = new Set()
            for (let i = 0; i < this.tabledata.length; i++) {
                filterUserData.add(this.tabledata[i].source)
                filterUserData.add(this.tabledata[i].target)

            }
            filterUserData = Array.from(filterUserData)

            function formatDate(date) {
                const year = date.getFullYear();
                const month = String(date.getMonth() + 1).padStart(2, '0');
                const day = String(date.getDate()).padStart(2, '0');
                const hours = String(date.getHours()).padStart(2, '0');
                const minutes = String(date.getMinutes()).padStart(2, '0');
                const seconds = String(date.getSeconds()).padStart(2, '0');
                const formattedDate = `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
                return formattedDate;
            }

            // 示例用法
            const date = new Date();
            const formattedDate = formatDate(date);

            let filterMesData = [];
            filterMesData.list = this.tabledata

            let timeData = new Set()
            // let timeInterval = Date.parse(filterMesData.list[1].time) - Date.parse(filterMesData.list[0].time)
            if (this.timeSlice == '') {
                this.$message.error("未输入时间片！")
                return
            }
            let timeInterval = this.timeSlice * 60000 * 10
            switch (timeInterval) {
                case 60000:
                    this.value = '分钟';
                    break;
                case 3600000:
                    this.value = '小时';
                    break;
                case 86400000:
                    this.value = '天';
                    break;
            }
            // this.value = timeInterval
            let maxTime = Date.parse(filterMesData.list[filterMesData.list.length - 1].time) + timeInterval
            let minTime = Date.parse(filterMesData.list[0].time)
            console.log((maxTime - minTime) / timeInterval);
            for (let i = 0; i < ((maxTime - minTime) / timeInterval); i++) {
                // console.log(filterMesData.list[i].source);
                timeData.add(formatDate(new Date(minTime + i * timeInterval)))
            }

            // console.log(filterUserData);
            timeData = Array.from(timeData)
            timeData.push(formatDate(new Date(Date.parse(timeData[timeData.length - 1]) + timeInterval)))
            // 86400000
            // console.log(timeData[0]);
            // console.log(filterUserData);





            // let filterUserData = [];
            // filterUserData = Object.values(store.state.filteruserres)
            // let filterUserDataNUM = [];
            // for(let i=0;i<filterUserData.length;i++){
            //     console.log(filterUserData[i].num);
            //     filterUserDataNUM.push(filterUserData[i].ids)
            // }



            // console.log('D3开始渲染');
            const svg = d3.select('#chart').attr('width', 3000);
            svg.select("#maingroup").remove();
            const width = +svg.attr('width');
            const height = +svg.attr('height');
            const margin = { top: 10, bottom: 30, left: 10, right: 700 };
            const innerwidth = width - margin.left - margin.right;
            const innerheight = height - margin.top - margin.bottom;
            // 初始化元素
            const g = svg.append('g')
                .attr('id', 'maingroup')
                .attr('transform', `translate(${margin.left},${margin.top})`);


            let background = g.append("rect").attr("class", "bg")
            let view = g.append("g").attr("class", "view")
            let linegroup = g.append("g").attr("class", "linegroup");
            let linegroup2 = g.append("g").attr("class", "linegroup2");
            let dotgroup = g.append("g").attr("class", "dotgroup");
            let grid = g.append("g").attr("class", "grid")
            let axis = g.append("g").attr("class", "axis")


            //     //在页面中添加svg 支持拖拽和缩放
            //    linegroup.attr("width", 3000).attr("height", 600)
            //         .call(d3.zoom().scaleExtent([1, 3]).on("zoom",
            //             function redraw(event) {
            //                 linegroup.attr("transform", d3.event.transform);
            //                 console.log(1111213);
            //             }
            //         ))
            //         // .attr("transform", "translate(" + 350 + "," + 20 + ")")
            //     // console.log(linegroup);


            // const timeX = []
            // for (let i = minTime.time; i < maxTime.time + 2; i++) {
            //     timeX.push(i)
            //     // console.log(i);
            // }
            // console.log(timeX);
            // 设置坐标轴
            const xScale = d3.scaleBand()
                .domain(timeData)
                .range([0, innerwidth]);
            const yScale = d3.scaleBand()
                .domain(filterUserData)
                .range([0, innerheight]);
            const yband = yScale.bandwidth()

            // 定义边
            let line = d3.line()
                .y(function (d) {
                    // let res;
                    // for(let i=0;i<filterUserData.length;i++){
                    //     if(filterUserData[i].name==d.name){
                    //         res = filterUserData[i].ids
                    //         // console.log(res);
                    //     }
                    // }
                    return yScale(d.name) + 0.5 * yband;
                })
                .x(function (d) {
                    return xScale(d.time);
                });


            // 定义箭头
            var defs = linegroup.append("defs");

            var arrowMarker = defs.append("marker")
                .attr("id", "arrow")
                .attr("markerUnits", "strokeWidth")
                .attr("markerWidth", "12")
                .attr("markerHeight", "12")
                .attr("viewBox", "0 0 12 12")
                .attr("refX", 5)
                .attr("refY", 0)
                .attr("orient", "auto")
            // .append('path')
            // .attr('fill', 'red')
            // .attr('d', 'M 0,-5 L 10,0 L 0,5');



            // filterMesDataByHold.forEach(d=>{
            //     d.source = filterUserData.find(e=>e.name == d.source).ids;
            //     d.target = filterUserData.find(e=>e.name == d.target).ids;
            // })
            // console.log(filterMesDataByHold);
            //绘制边和箭头
            filterMesDataByHold.forEach(d => {
                // console.log(d.source, d.time);
                linegroup.append('path')
                    .attr('d', line([{
                        name: d.source,
                        time: formatDate(new Date(d.time))
                    }, {
                        name: d.target,
                        time: formatDate(new Date(Date.parse(d.time) + timeInterval))
                    }]))
                    .attr('id', `E${d.id}`)
                    .style('cursor', 'pointer')
                    .classed("chooseline", false)
                    .classed("unchooseline", true)
                    // .attr('class', `M${d.markov}`)
                    // .attr('fill', 'none')
                    .attr('stroke-width', 5)
                    .attr("marker-end", "url(#arrow)")
                    // .style("stroke", that.messageColor)
                    // .style("stroke-dasharray", 6)
                    .on("click", function () {

                        if (that.filterresFromUser.find(user => user == d.id) == null) {
                            that.$message.success("选取成功" + d.id);
                            that.filterresFromUser.push(d.id)
                            d3.select(this)
                                .classed("chooseline", true)
                                .classed("unchooseline", false)
                        } else {
                            that.$message.error("取消选取" + d.id);
                            let index = that.filterresFromUser.indexOf(d.id);
                            that.filterresFromUser.splice(index, 1)
                            d3.select(this)
                                .classed("chooseline", false)
                                .classed("unchooseline", true)
                        }

                    })
                    .append('title')
                    .text(dd => {
                        return `source: ${d.source}\ntarget: ${d.target}\ntime: ${d.time}\ncontent: ${d.content}`;
                    });;

                // d3.select(`#E${44}`)
                // .classed("chooseline",true)
                // .classed("unchooseline",false)

                // 绘制点
                dotgroup.append("circle")
                    .attr("class", `T${d.source}`)
                    .attr("cy", yScale(d.source) + 0.5 * yband)
                    .attr("cx", xScale(formatDate(new Date(d.time))))
                    .attr("r", 8)
                    .style("fill", function () {
                        if (IPfromChoose != []) {
                            for (let i = 0; i < IPfromChoose.length; i++) {
                                if (isIPInNetwork(d.source, IPfromChoose[i])) {
                                    return "red"
                                }
                            }
                        }else{
                            return "blue"
                        }
                    });

                dotgroup.append("circle")
                    .attr("class", `T${d.target}`)
                    .attr("cy", yScale(d.target) + 0.5 * yband)
                    .attr("cx", xScale(formatDate(new Date(Date.parse(d.time) + timeInterval))))
                    .attr("r", 8)
                    .style("fill", function () {
                        if (IPfromChoose != []) {
                            for (let i = 0; i < IPfromChoose.length; i++) {
                                if (isIPInNetwork(d.target, IPfromChoose[i])) {
                                    return "red"
                                }
                            }
                        }else{
                            return "blue"
                        }
                    });

            });

            // //连线
            // filterresFromUser = []
            for (let i = 0; i < filterMesDataByHold.length; i++) {
                if (filterMesDataByHold.find(e =>
                    (e.target == filterMesDataByHold[i].source && Date.parse(e.time) >= Date.parse(filterMesDataByHold[i].time) - this.formInline.threshold * timeInterval)
                    || (e.source == filterMesDataByHold[i].target && Date.parse(e.time) <= Date.parse(filterMesDataByHold[i].time) + this.formInline.threshold * timeInterval)
                ) != null) {
                    this.filterresFromUser.push(filterMesDataByHold[i].id);
                    // console.log(filterMesDataByHold[i].id);
                    d3.select(`#E${filterMesDataByHold[i].id}`)
                        .classed("chooseline", true)
                        .classed("unchooseline", false)

                    // console.log(store.state.filtermesresByhold[i]);
                    // linegroup2
                    //     .append('path')
                    //     .attr('d', line([{
                    //         name: store.state.filtermesresByhold[i].source.name,
                    //         time: store.state.filtermesresByhold[i].time
                    //     }, {
                    //         name: store.state.filtermesresByhold[i].target.name,
                    //         time: store.state.filtermesresByhold[i].time + 1
                    //     }]))
                    //     .attr('id', `E2${store.state.filtermesresByhold[i].id}`)
                    //     .style("stroke", that.marColor)
                    //     .style("stroke-dasharray", 0)
                    //     // .attr('class', `M${d.markov}`)
                    //     .attr('fill', 'none')
                    //     .attr('stroke-width', 5)
                    //     .attr("marker-end", "url(#arrow)")
                    //     .style('cursor', 'pointer')
                    //     .on("click", function () {
                    //         ElMessage.error("取消选取" + store.state.filtermesresByhold[i].id);
                    //         let index = store.state.filterresFromUser.indexOf(store.state.filtermesresByhold[i].id);
                    //         store.state.filterresFromUser.splice(index, 1)
                    //         // console.log("取消" + store.state.filterresFromUser);
                    //         d3.select(`#E2${store.state.filtermesresByhold[i].id}`)
                    //             .remove()
                    //     })
                    // d3.select(`#E${store.state.filtermesresByhold[i].id}`)
                    //     .classed("chooseline", true)
                    //     .classed("unchooseline", false)
                }
            }

            // let linkmust = linegroup2
            //     .append('path')
            //     .attr('d', line([{
            //         name: "192.168.1.2",
            //         time: 1
            //     }, {
            //         name: "192.168.1.10",
            //         time: 2
            //     }]))
            //     // .attr('id', `E${d.id}`)
            //     .style("stroke", "red")
            //     .style("stroke-dasharray", 0)
            //     // .attr('class', `M${d.markov}`)
            //     .attr('fill', 'none')
            //     .attr('stroke-width', 5)
            //     .attr("marker-end", "url(#arrow)")



            // //加一个坐标轴的遮罩层
            let xAxisModel = svg.select('#maingroup')
                .select('.axis')
                .append('rect')
                .attr('x', -100)
                .attr('y', -50)
                .attr('width', '100%')
                .attr('height', 60)
                .attr('fill', 'white')
            let yAxisModel = svg.select('#maingroup')
                .select('.axis')
                .append('rect')
                .attr('x', -140)
                .attr('y', -30)
                .attr('width', 130)
                .attr('height', '100%')
                .attr('fill', 'white')




            const yAxis = d3.axisLeft(yScale)
            const gY = axis.append('g')
                .classed('yAxis', true)
                .call(yAxis)
            gY.selectAll(".tick")
                .each(function (d, i) {
                    d3.select(this).attr("id", "tickIP-" + d);
                    if (IPfromChoose != []) {
                        for (let i = 0; i < IPfromChoose.length; i++) {
                            if (isIPInNetwork(d, IPfromChoose[i])) {
                                d3.select(this).style("color", "red")
                            }
                        }
                    }
                    // var result_ip = ipAddress.match(/^(\d+\.\d+\.\d+)/);
                    // d3.select(this).style("color", "red")
                })

            const xAxis = d3.axisTop(xScale);
            const gX = axis.append('g')
                .classed('xAxis', true)
                .call(xAxis);
            gX.selectAll(".tick")
                .each(function (d, i) {
                    // 为每个tick生成唯一的id
                    d3.select(this).attr("id", "tick-" + i);
                    if (i % that.accY != 0) { d3.select(this).style("display", "none");; }
                });



            function calAppendY() {
                let appendMove = 0;
                gX.selectAll(".tick").each(function (d) {
                    let text_width = this.getBoundingClientRect().height
                    if (appendMove < text_width)
                        appendMove = text_width
                })
                return appendMove
            }
            function calAppendX() {
                let appendMove = 0;
                gY.selectAll(".tick").each(function (d) {
                    let text_width = this.getBoundingClientRect().width
                    if (appendMove < text_width)
                        appendMove = text_width
                })

                return appendMove + 60
            }

            gY.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            gX.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            yAxisModel.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            xAxisModel.attr("transform", `translate(${calAppendX()},${calAppendY()})`)

            svg.select('.linegroup').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)
            svg.select('.linegroup2').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)
            svg.select('.dotgroup').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)

            //配置字体大小
            svg.select('.xAxis').selectAll('.tick').selectAll('text').style('font-size', 17)
            svg.select('.yAxis').selectAll('.tick').selectAll('text').style('font-size', 17)


            const zoom = d3.zoom()
                .scaleExtent([0.5, 40])
                .translateExtent([[-1000, -1000], [width + 9000, height + 1000]])
                // .filter(filter)
                .on("zoom", zoomed);


            svg.call(zoom)



            function zoomed() {
                const radio = 0.5
                svg.select('.linegroup').attr("transform", d3.event.transform);
                svg.select('.linegroup2').attr("transform", d3.event.transform);
                svg.select('.dotgroup').attr("transform", d3.event.transform);
                // svg.select('.linegroup').attr("transform",`translate(${d3.event.transform.x},${d3.event.transform.y}) scale(${d3.event.transform.k})`);
                // svg.select('.dotgroup').attr("transform",`translate(${d3.event.transform.x},${d3.event.transform.y}) scale(${d3.event.transform.k})`);

                //连线的粗细随缩放保持不变
                svg.select('.linegroup').selectAll('path').attr("stroke-width", 5 * 1.0 / d3.event.transform.k)
                //连线的粗细随缩放保持不变
                svg.select('.linegroup2').selectAll('path').attr("stroke-width", 5 * 1.0 / d3.event.transform.k)
                //点的大小随缩放保持不变
                svg.select('.dotgroup').selectAll('circle').attr("r", `${8 * 1.0 / d3.event.transform.k}`)
                //轴的字体大小，刻度线大小，轴的粗细保持不变
                svg.select('.yAxis').selectAll('.tick').selectAll('text').attr("transform", `scale(${1.0 / d3.event.transform.k})`)
                svg.select('.yAxis').selectAll('.tick').selectAll('line').attr("transform", `scale(${1.0 / d3.event.transform.k})`)
                svg.select('.yAxis').select('path').style('stroke-width', 2.0 / d3.event.transform.k + 'px')
                svg.select('.xAxis').selectAll('.tick').selectAll('text').attr("transform", `scale(${1.0 / d3.event.transform.k})`)
                svg.select('.xAxis').selectAll('.tick').selectAll('line').attr("transform", `scale(${1.0 / d3.event.transform.k})`)
                svg.select('.xAxis').select('path').style('stroke-width', 2.0 / d3.event.transform.k + 'px')

                //位移，保持相对位置
                svg.select('.xAxis').attr("transform", `translate(${d3.event.transform.x},${calAppendY()}) scale(${d3.event.transform.k})`)
                svg.select('.yAxis').attr("transform", `translate(${calAppendX() - 50},${d3.event.transform.y}) scale(${d3.event.transform.k})`)

            }
        },
    },
    mounted() {
        var ipAddress = "102.168.1.0";
        var result = ipAddress.match(/^(\d+\.\d+\.\d+)/);
        if (result) {
            console.log(result[1] + '.');
        } else {
            console.log("无法提取IP地址的前三个部分");
        }
        // let a = '2023-8-27  00:00:00'

        // function formatDate(date) {
        //     const year = date.getFullYear();
        //     const month = String(date.getMonth() + 1).padStart(2, '0');
        //     const day = String(date.getDate()).padStart(2, '0');
        //     const hours = String(date.getHours()).padStart(2, '0');
        //     const minutes = String(date.getMinutes()).padStart(2, '0');
        //     const seconds = String(date.getSeconds()).padStart(2, '0');

        //     const formattedDate = `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
        //     return formattedDate;
        // }

        // console.log(formatDate(new Date(Date.parse(a) + 86400000 * 365)));
        // // 示例用法
        // const date = new Date();
        // const formattedDate = formatDate(date);
        // console.log(formatDate(new Date(Date.parse(formattedDate) + 86400000)));
    },
    // updated() {
    //     d3.select('#maingroup').remove();
    //     this.generateVis2();
    //     console.log(this.selectvalue);
    //     console.log(this.currentRow);
    // },
};
</script>
<style>
.mar-title {
    display: flex;
    color: white;
    background-color: #16C6FC;
}

.input-box {
    display: flex;
    flex-direction: row;
}

.container {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    width: 2936px;
    height: 500px;

}

.page {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
}

.control {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 5%;
    padding-right: 30px;
}

.view1 {
    width: 100%;
}

.unchooseline {
    stroke: #DCDCDC;
    stroke-dasharray: 6;
}

.chooseline {
    stroke: black;
    stroke-dasharray: 0;
}
</style>
  