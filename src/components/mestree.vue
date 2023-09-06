<template>
    <div>
        <div class="mar-title">
            <h2 style="margin: 5px 5px;font-size: 20px;">消息流</h2>
            <el-button @click="importall" style="border-radius: 10px; margin-left: 10px;">导入</el-button>
            <el-button @click="exportall" style="border-radius: 10px; margin-left: 10px;">导出</el-button>
            <el-dialog :visible.sync="dialogVisble" width="500px" :before-close="handleClose">
                <div>时间片：
                    <el-input v-model="timeSlice" placeholder="请输入" style="width: 20%;">
                    </el-input>
                    *10分钟
                </div>
                <el-upload action="#" :http-request="uploadFileTest" :limit="1" :drag=true style="margin-top: 10px;">
                    <i class="el-icon-upload"></i>
                    <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
                </el-upload>
            </el-dialog>
        </div>
        <div>
            <el-tabs v-model="activeName">
                <el-tab-pane label="消息流" name="first" style="display: flex;flex-direction: column;">
                    <div style="display: flex;flex-direction: row;">
                        <el-input v-model="accY" placeholder="纵轴精确度" style="width: 10%;margin-right: 10px;"></el-input>
                        <el-button @click="generateVis2"
                            style="border-radius: 10px;  background-color: aquamarine; color: blue;">绘制</el-button>
                            <el-button type="primary" icon="el-icon-edit" @click="openAcc" circle></el-button>
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
                        </el-form-item>
                        <el-form-item>
                            <el-button type="primary" @click="onSubmit">查询</el-button>
                        </el-form-item>
                    </el-form>
                    <el-table :data="tableDataForShow">
                        <el-table-column prop="id" label="ID" width="55" align="center"></el-table-column>
                        <el-table-column prop="source" label="起始点"></el-table-column>
                        <el-table-column prop="target" label="目标点">
                        </el-table-column>
                        <el-table-column prop="time" label="传递时间"></el-table-column>
                    </el-table>
                    <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange"
                        :current-page=page.index :page-sizes="[5, 10, 15]" :page-size=page.size
                        layout="total, sizes, prev, pager, next, jumper" :total=page.total>
                    </el-pagination>
                </el-tab-pane>
            </el-tabs>
        </div>
    </div>
</template>
<script>
import * as d3 from 'd3';
import ElementUI from 'element-ui';
import Vue from 'vue'
import 'element-ui/lib/theme-chalk/index.css';
import axios from "axios"

Vue.use(ElementUI);

export default {
    name: 'mestree',
    data() {
        return {
            activeName: "first",//默认第一页        
            page: {//分页参数
                size: 10,
                index: 1,
                total: 0
            },
            formInline: {//表单数据
                user: '',//IP
                hop: '',//跳数
                datastart: '',//起始时间
                dataend: '',//终止时间
                threshold: 1,//阈值
                timestart: '',//换算成毫秒的时间
                timeend: '',//换算成毫秒的时间
                thresholdByTime: '',//换算成毫秒的阈值
            },
            filename: "无",//文件名称
            dialogVisble: false,//对话框可视
            tabledata: null,//导入的全部数据
            tableDataForShow: null,//表格展示的已分完页数据
            accY: "",//纵轴准确度
            filterresFromUser: [],//用户选择好的数据，即导出数据
            timeSlice: "",//用户自己填写的时间片
            timeAll:""//一共有多少时间点，为了做省略
        };
    },
    methods: {
        importall() {
            //导入，显示导入对话框
            this.dialogVisble = true
        },
        handleClose() {
            //关闭导入对话框
            this.dialogVisble = false
        },
        exportall() {
            //导出，数据为this.filterresFromUser，后续如何使用数据就不知道了
            console.log(this.filterresFromUser);
        },
        async uploadFileTest(item) {
            //导入文件，先保存文件到本地，再读取，如果数据不需要导入，可以注释掉
            let formdataw = new FormData()
            formdataw.append('file', item.file)
            //保存文件至本地
            await axios({
                url: 'http://127.0.0.1:5002/mesBE/upload',
                method: "POST",
                data: formdataw
            }).then(res => {
                this.filename = res.data
            })
            //读取文件
            await axios({
                url: 'http://127.0.0.1:5002/mesBE/read',
                method: "POST",
                data: { name: this.filename }
            }).then(res => {
                this.tabledata = res.data
            })
            //数据进行时间处理，增添数据属性，把2023-8-22 00:00:00 转换成毫秒，方便后续画图比较
            for (let i = 0; i < this.tabledata.length; i++) {
                this.tabledata[i].timesecond = Date.parse(this.tabledata[i].time)
            }
            //数据导入结束，表格数据展示，分页
            this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
            this.page.total = this.tabledata.length
        },
        async onSubmit() {
            //数据检索，提交数据和表单数据到后台进行检索
            //先把时间改成毫秒
            this.formInline.timestart = Date.parse(this.formInline.datastart)
            this.formInline.timeend = Date.parse(this.formInline.dataend)
            if (this.formInline.user == '') {
                this.$message.error("请输入IP")
            } else if (this.formInline.hop == '') {
                this.$message.error("请输入跳数")
            } else if (this.formInline.datastart == '') {
                this.$message.error("请输入起始时间")
            } else if (this.formInline.dataend == '') {
                this.$message.error("请输入终止时间")
            } else {
                //传入后台
                await axios({
                    url: 'http://127.0.0.1:5002/mesBE/searchData',
                    method: "POST",
                    data: {
                        data: this.tabledata,
                        form: this.formInline
                    }
                }).then(res => {
                    this.tabledata = res.data
                })
                //检索完毕，数据分页
                this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
                this.page.total = this.tabledata.length
                //把原始数据的作图给删掉
                d3.select("#maingroup").remove()
            }
        },
        handleSizeChange(val) {
            //分页修改每页条数
            // console.log(`每页 ${val} 条`);
            this.page.size = val
            this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
        },
        handleCurrentChange(val) {
            //分页修改当前页数
            // console.log(`当前页: ${val}`);
            this.page.index = val
            this.tableDataForShow = this.tabledata.slice((this.page.index - 1) * this.page.size, (this.page.index) * this.page.size)
        },
        openAcc(){
            //直接修改坐标轴精确度，不用重新绘制
            for(let i = 0;i<this.timeAll;i++){
                if (i % this.accY != 0) { d3.select(`#tick-${i}`).style("display", "none")}
                else{ d3.select(`#tick-${i}`).style("display", "")}
            }
        },
        generateVis2() {
            //作图
            //若无导入数据按钮，就直接导入json格式数据，如下
            // this.tabledata =
            //     [
            //         {
            //             "id": 1,
            //             "source": "192.168.1.1",
            //             "target": "192.168.1.2",
            //             "time": "2023-08-23 00:00"
            //         },
            //         {
            //             "id": 2,
            //             "source": "192.168.1.3",
            //             "target": "192.168.1.4",
            //             "time": "2023-08-23 00:10"
            //         },
            //         {
            //             "id": 3,
            //             "source": "192.168.1.5",
            //             "target": "192.168.1.6",
            //             "time": "2023-08-23 00:20"
            //         },
            //         {
            //             "id": 4,
            //             "source": "192.168.1.7",
            //             "target": "192.168.1.8",
            //             "time": "2023-08-23 00:30"
            //         },
            //         {
            //             "id": 5,
            //             "source": "192.168.1.9",
            //             "target": "192.168.1.10",
            //             "time": "2023-08-23 00:40"
            //         },
            //         {
            //             "id": 6,
            //             "source": "192.168.1.11",
            //             "target": "192.168.1.12",
            //             "time": "2023-08-23 00:50"
            //         },
            //         {
            //             "id": 7,
            //             "source": "192.168.1.13",
            //             "target": "192.168.1.14",
            //             "time": "2023-08-23 01:00"
            //         },
            //         {
            //             "id": 8,
            //             "source": "192.168.1.15",
            //             "target": "192.168.1.16",
            //             "time": "2023-08-23 01:10"
            //         },
            //         {
            //             "id": 9,
            //             "source": "192.168.1.17",
            //             "target": "192.168.1.18",
            //             "time": "2023-08-23 01:20"
            //         },
            //         {
            //             "id": 10,
            //             "source": "192.168.1.19",
            //             "target": "192.168.1.20",
            //             "time": "2023-08-23 01:30"
            //         }
            //     ]


            //纵轴精确度若没填，默认1
            if (this.accY == '') {
                this.accY = 1
            }
            let that = this;

            //对数据先进行处理

            //把form表单中 IP段；IP段 分割
            let IPfromChoose = []
            if (this.formInline.user != "") {
                IPfromChoose = this.formInline.user.split(";")
                console.log(IPfromChoose);
            }

            //判断ip是否是该network ip段
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

            //绘制的数据
            let filterMesDataByHold = this.tabledata;

            //所有ip数据，去重
            let filterUserData = new Set()
            for (let i = 0; i < this.tabledata.length; i++) {
                filterUserData.add(this.tabledata[i].source)
                filterUserData.add(this.tabledata[i].target)
            }
            filterUserData = Array.from(filterUserData)

            //定义时间格式
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
            // const date = new Date();
            // const formattedDate = formatDate(date);

            //计算时间片
            let timeData = new Set()
            //默认导入数据时间片，但是现在为用户输入时间片，所以用不到
            // let timeInterval = Date.parse(filterMesData.list[1].time) - Date.parse(filterMesData.list[0].time)
            if (this.timeSlice == '') {
                this.$message.error("未输入时间片！")
                return
            }
            //时间片单位是10分钟
            let timeInterval = this.timeSlice * 60000 * 10
            //计算最大时间
            let maxTime = Date.parse(this.tabledata[this.tabledata.length - 1].time) + timeInterval
            //计算最小时间
            let minTime = Date.parse(this.tabledata[0].time)
            //timedata是横轴的数据，需要补充上没有时间的数据，所以用最大最小时间来
            for (let i = 0; i < ((maxTime - minTime) / timeInterval); i++) {
                timeData.add(formatDate(new Date(minTime + i * timeInterval)))
            }
            timeData = Array.from(timeData)
            //作图逻辑原因，需要补充最大后面一个时间
            timeData.push(formatDate(new Date(Date.parse(timeData[timeData.length - 1]) + timeInterval)))
            this.timeAll = timeData.length

            //绘图
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

            //绘制边和箭头
            filterMesDataByHold.forEach(d => {
                linegroup.append('path')
                    .attr('d', line([{
                        name: d.source,
                        time: formatDate(new Date(d.time))//格式化时间
                    }, {
                        name: d.target,
                        time: formatDate(new Date(Date.parse(d.time) + timeInterval))//格式化下一个时间
                    }]))
                    .attr('id', `E${d.id}`)//唯一id
                    .style('cursor', 'pointer')
                    .classed("chooseline", false)//选取样式
                    .classed("unchooseline", true)//未选取样式
                    .attr('stroke-width', 5)
                    .attr("marker-end", "url(#arrow)")
                    .on("click", function () {
                        //点击选取，把边加到结果里面去
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
                    //显示边数据
                    .append('title')
                    .text(dd => {
                        return `source: ${d.source}\ntarget: ${d.target}\ntime: ${d.time}`;
                    });;

                // 绘制点
                dotgroup.append("circle")
                    .attr("class", `T${d.source}`)
                    .attr("cy", yScale(d.source) + 0.5 * yband)
                    .attr("cx", xScale(formatDate(new Date(d.time))))
                    .attr("r", 8)
                    .style("fill", function () {
                        //对筛选条件进行判断，若是筛选的标红
                        if (IPfromChoose != []) {
                            for (let i = 0; i < IPfromChoose.length; i++) {
                                if (isIPInNetwork(d.source, IPfromChoose[i])) {
                                    return "red"
                                }
                            }
                        } else {
                            return "blue"
                        }
                    });

                dotgroup.append("circle")
                    .attr("class", `T${d.target}`)
                    .attr("cy", yScale(d.target) + 0.5 * yband)
                    .attr("cx", xScale(formatDate(new Date(Date.parse(d.time) + timeInterval))))
                    .attr("r", 8)
                    .style("fill", function () {
                        //对筛选条件进行判断，若是筛选的标红
                        if (IPfromChoose != []) {
                            for (let i = 0; i < IPfromChoose.length; i++) {
                                if (isIPInNetwork(d.target, IPfromChoose[i])) {
                                    return "red"
                                }
                            }
                        } else {
                            return "blue"
                        }
                    });


            });

            //通过时间阈值对原始点进行连线
            for (let i = 0; i < filterMesDataByHold.length; i++) {
                // 若在该时间阈值内，存在一条消息的起点是我的终点（我的时间之后），或者一条消息的终点是我的起点（我的时间之前），认为我们是连接一起的，消息标黑
                if (filterMesDataByHold.find(e =>
                    (e.target == filterMesDataByHold[i].source && Date.parse(filterMesDataByHold[i].time) - this.formInline.threshold * timeInterval <= Date.parse(e.time) <= Date.parse(filterMesDataByHold[i].time))
                    || (e.source == filterMesDataByHold[i].target && Date.parse(filterMesDataByHold[i].time) <= Date.parse(e.time) <= Date.parse(filterMesDataByHold[i].time) + this.formInline.threshold * timeInterval)
                ) != null) {
                    this.filterresFromUser.push(filterMesDataByHold[i].id);
                    d3.select(`#E${filterMesDataByHold[i].id}`)
                        .classed("chooseline", true)
                        .classed("unchooseline", false)
                }
            }

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


            // 坐标绘制
            // y轴
            const yAxis = d3.axisLeft(yScale)
            const gY = axis.append('g')
                .classed('yAxis', true)
                .call(yAxis)
            gY.selectAll(".tick")
                .each(function (d, i) {
                    d3.select(this).attr("id", "tickIP-" + d);
                    // 若该点被选到标红
                    if (IPfromChoose != []) {
                        for (let i = 0; i < IPfromChoose.length; i++) {
                            if (isIPInNetwork(d, IPfromChoose[i])) {
                                d3.select(this).style("color", "red")
                            }
                        }
                    }
                })

            // x轴
            const xAxis = d3.axisTop(xScale);
            const gX = axis.append('g')
                .classed('xAxis', true)
                .call(xAxis);
            gX.selectAll(".tick")
                .each(function (d, i) {
                    d3.select(this).attr("id", "tick-" + i);
                    // 该点准确度不为0，省略
                    if (i % that.accY != 0) { d3.select(this).style("display", "none") }
                });



            function calAppendY() {
                // y轴移动
                let appendMove = 0;
                gX.selectAll(".tick").each(function (d) {
                    let text_width = this.getBoundingClientRect().height
                    if (appendMove < text_width)
                        appendMove = text_width
                })
                return appendMove
            }
            function calAppendX() {
                // x轴移动
                let appendMove = 0;
                gY.selectAll(".tick").each(function (d) {
                    let text_width = this.getBoundingClientRect().width
                    if (appendMove < text_width)
                        appendMove = text_width
                })

                return appendMove + 60
            }

            // 坐标轴和遮罩层移动
            gY.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            gX.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            yAxisModel.attr("transform", `translate(${calAppendX()},${calAppendY()})`)
            xAxisModel.attr("transform", `translate(${calAppendX()},${calAppendY()})`)

            // 图移动
            svg.select('.linegroup').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)
            svg.select('.linegroup2').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)
            svg.select('.dotgroup').attr("transform", `translate(${calAppendX()},${calAppendY() + 10})`)

            //配置字体大小
            svg.select('.xAxis').selectAll('.tick').selectAll('text').style('font-size', 17)
            svg.select('.yAxis').selectAll('.tick').selectAll('text').style('font-size', 17)


            const zoom = d3.zoom()
                .scaleExtent([0.5, 40])
                .translateExtent([[-1000, -1000], [width + 9000, height + 1000]])
                .on("zoom", zoomed);


            svg.call(zoom)



            // 缩放
            function zoomed() {
                const radio = 0.5
                // 图缩放
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
        // var ipAddress = "102.168.1.0";
        // var result = ipAddress.match(/^(\d+\.\d+\.\d+)/);
        // if (result) {
        //     console.log(result[1] + '.');
        // } else {
        //     console.log("无法提取IP地址的前三个部分");
        // }
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
  