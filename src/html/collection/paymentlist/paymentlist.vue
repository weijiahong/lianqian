<template>
     <div class='container'>
         <div class="animate">
            <form class="layui-form" id="form">
                <div class="layui-form-item">
                    <div class="layui-inline">
                        <label class="layui-form-label">订单号</label>
                        <div class="layui-input-inline">
                            <input type="text" name="orderId" @input="change($event.srcElement ? $event.srcElement : $event.target)" placeholder="请输入订单号" autocomplete="off" class="layui-input">
                        </div>
                    </div>
                    <div class="layui-inline">
                        <label class="layui-form-label" style="width:85px;">订单创建时间</label>
                        <div class="layui-input-inline" style="margin-right:0;">
                            <input class="layui-input" readonly  name="createStartTime" placeholder="请选择" id="start">
                        </div>
                        <label class="layui-form-label" style="width:auto">至</label>
                        <div class="layui-input-inline">
                            <input class="layui-input" readonly name="createEndTime" placeholder="请选择" id="end">
                        </div>
                    </div>
                    <div class="layui-inline">
                        <button class="layui-btn layui-btn-small layui-btn-normal layui-btn-custom"  lay-submit lay-filter="seach" style="background:#37a3ff;color:#fff;border-radius:2px;">
                            <i class="layui-icon" style="display:inline-block;transform:scale(0.9);">&#xe615;</i> 查询
                        </button>
                        <button type="reset" class="layui-btn layui-btn-small layui-btn-normal layui-btn-custom"  @click="getTable()">
                            <i class="layui-icon">&#xe631;</i> 重置
                        </button>
                    </div>
                </div>
            </form>
            <form class="layui-form">
                <div class="layui-form-item" style="padding-top:10px;">
                    <div class="layui-inline">
                        <label class="layui-form-label" style="width:auto;padding-left:0px;">选中订单标记为</label>
                        <div class="layui-input-inline" style="width:110px;">
                                <select name="enabled" lay-verify="required">
                                    <option value="2">已还款</option>
                                    <option value="0">驳回</option>
                                </select>
                        </div>
                    </div>
                    <div class="layui-inline">
                        <button class="layui-btn layui-btn-small layui-btn-normal layui-btn-custom" lay-submit lay-filter="commit_allot">
                            <i class="layui-icon">&#xe605;</i> 确认
                        </button>
                    </div>
                </div>
                <table class="layui-table" id="table">
                    <colgroup>
                    </colgroup>
                    <thead>
                        <th><input type="checkbox" name="" lay-skin="primary" lay-filter="allChoose"></th>
                        <th>序号</th>
                        <th>操作</th>
                        <th>订单号</th>
                        <th>订单状态</th>
                        <th>创建时间</th>
                        <th>欠款人</th>
                        <th>证件号</th>
                        <th>欠款</th>
                        <th>催收人</th>
                    </thead>
                    <tbody>
                        <tr  v-for="(item,index) in list">
                            <td><input type="checkbox" :name="index" lay-skin="primary" :value="item.collectionId"></td>
                            <td v-if="pageIndex-1==0" v-text="index+1"></td>
                            <td v-else v-text="pageSize*(pageIndex-1) + index + 1"></td>
                            <td>
                                <div class="layui-btn layui-btn-small layui-btn-normal layui-btn-custom radius edit" @click="go_url(item)">查看</div>
                            </td>
                            <td v-text="item.orderId">订单号</td>
                            <td v-text="item.collectionSts">订单状态</td>
                            <td v-text="item.createTime">创建时间</td>
                            <td v-text="item.borrowerName">欠款人</td>
                            <td>{{item.idNumber | ellipsisId}}</td>
                            <td v-text="item.repayAmout ? (item.repayAmout/100).toFixed(2) : '0.00'">欠款</td>
                            <td v-text="item.dealUserName">催收人</td>
                        </tr>
                    </tbody>
                </table>
            </form>
            <div id="pagination" style="text-align:right"></div>
            <div v-show="!list" style="text-align:center">暂无数据</div>
        </div>
    </div>
</template>
<script>
import {verify, date, page, Common, checkbox} from 'js/layuiCommon';
import request from 'js/interface/request';
import {tab, winLayer} from 'js/common';
export default {
    data () {
        return {
            roleName: '', //登录人权限  admin 管理员 director 催收总监 manager 催收经理 supervisor主管 staff 员工
            //0代表没有排序， 1代表正序 2代表倒叙
            first: true,
            pageSize: 10, //一页多少条
            pageIndex: 1, // 当前页码
            group: '', //组数据
            staff: '', // 员工数据
            form_table: '', //点击查询时存储查询数据
            //查询排序;
            //订单创建时间:'create_time'
            //欠款：'repay_amout'
            //应还款时间：'repay_date'
            //订单倒计时：待定最后
            //催收时间：'modify_time'
            orderBy: 'create_time', //默认排序按订单创建时间正序
            dir: 'asc', // asc正序 desc倒序
            stsCollectionTagsRecordList: [], // 关系标签
            list: '' //表格数据
        }
    },
    computed: {
    },
    created () {
        //登录人权限  admin 管理员 director 催收总监 manager 催收经理 supervisor主管 staff 员工
        this.roleName = window.localStorage.getItem('roleName'); // 获取职位
    },
    mounted () {
         //初始化
        this.init();
        this.initialize();
    },
    methods: {
        //初始化
        init () {
            //表单搜索
            verify.form('seach', (data) => {
                let data1 = data.field;
                 // endRepayDate startRepayDate 还款时间
                if (data1.createStartTime&&data1.createEndTime) {
                    if (data1.createStartTime > data1.createEndTime) {
                        Common.alert('开始时间必须小于等于结束时间')
                        return false;
                    }
                }
                // 点击查询页码赋值为1；
                data1.pageIndex = 1;
                data1.pageSize = this.pageSize;
                this.first = true;
                this.getTable(data1);
                return false;
            });
             //调整分配人员
            verify.form('commit_allot', (data) => {
                let data1 = data.field;
                this.orderData(data1); //协催订单分配数据处理
            });
            date.date_input('start', 'end'); // 还款时间
            this.getTable(); //获取表格数据
            checkbox.checkAll('table'); //表格全选
        },
        //表格数据
        getTable (data) {
            window.$('#table input').prop('checked', false);
            let tableDate = ''
            if (data) {
                tableDate = data;
            } else {
                tableDate = {
                    pageIndex: 1,
                    pageSize: 10
                }
                this.first = true;
                this.pageSize = 10;
                // console.log('初始化表格')
            }
            this.form_table = tableDate; // 存储数据;
            console.log(tableDate)
            request('/collection/getPreCompleteCollection.html', tableDate, (response) => {
                console.log(response.data.data)
                if (response.data.respCode === '000000') {
                    let output = response.data.data;
                    if (output) {
                        this.list = output.collectionPreCompleteRecordList; // 数据list
                        this.pageIndex = tableDate.pageIndex; //成功后页码赋值
                    } else {
                        return false;
                    }
                    //分页第一次和搜索时要更新页码
                    if (this.first) {
                        this.first = false;
                        this.addPagination(output.page, tableDate, output.totalCount);// //总页码赋值
                    }
                    this.initialize(); // 初始化控件
                } else {
                    Common.alert(response.data.respMsg);
                }
            }, (e) => {
                Common.alert(e);
            })
        },
        //添加分页
        addPagination (pages, obj, totalCount) {
            let config = {
                that: this,
                pagination: 'pagination',
                pages: pages,
                pageIndexName: 'pageIndex',
                pageSizeName: 'pageSize',
                firstName: 'first',
                requestMethod: 'getTable',
                totalCount: totalCount
            }
            page.table_laypage(config, obj);
        },
        //排序
        sortData (item, index) {
            for (let i = 0; i < this.thData.length; i++) {
                if (i === index) {
                    if (item.sort === 1) {
                        item.sort = 2;
                        this.form_table.dir = 'desc'; //倒叙
                    } else {
                        item.sort = 1;
                        this.form_table.dir = 'asc'; // 正序
                    }
                } else {
                    this.thData[i].sort = 1;
                }
            }
            this.form_table.pageIndex = 1; // 页码重置为1
            this.form_table.orderBy = item.orderBy; // 排序的名字
            console.log(this.form_table)
            this.first = true;
            this.getTable(this.form_table); //排序更新数据
        },
        //订单分配的数据处理
        orderData (data) {
            let enabled = data.enabled; //保存
            delete data.enabled; // 删除属性只要订单号
            let arr = Object.values(data); //只取value值
            //判断至少有一个订单号
            // console.log(arr)
            if (arr.length <= 0) {
                Common.alert('至少选择一个订单！');
                return false;
            }
            winLayer.loadOn(2) // 开始加载
            let obj = {
                enabled: enabled
            };
            let collectionIdRecord = [];
            for (let i = 0; i < arr.length; i++) {
                let obj = {
                    collectionId: arr[i]
                }
                collectionIdRecord.push(obj)
            }
            obj.collectionIdRecord = collectionIdRecord;
            this.order_allocation(obj); //分配订单
        },
        //订单分配提交
        order_allocation (data) {
            console.log(data)
            request('/collection/collectionPreCompleteDeal.html', data, (response) => {
                console.log(response.data.data)
                if (response.data.respCode === '000000') {
                    Common.alert(response.data.respMsg);
                    window.$('#table input[type="checkbox"]').prop('checked', false);
                    this.first = true; //更新分页
                    this.getTable(this.form_table);
                    winLayer.loadOff() // 关闭load
                } else {
                    winLayer.loadOff() // 关闭load
                    Common.alert(response.data.respMsg);
                }
            }, (e) => {
                winLayer.loadOff() // 关闭load
                Common.alert(e);
            })
        },
        //跳转协催
        go_url (item) {
            tab.tabAdd('订单详情', './collection/detailorder.html?collectionId=' + item.collectionId);
        },
         //去空格
        change (that) {
            that.value = that.value.trim();
        },
        //layui初始化控件（要多次初始化）
        initialize () {
            this.$nextTick(() => {
                window.layui.use('form', () => {
                    var form = window.layui.form();
                    form.render();
                })
            })
        }
    },
    filters: {
        // 身份证省略显示
        ellipsisId: function (value) {
            if (!value) {
                return;
            }
            var str = value.substring(0, 4);
            for (var i = 4; i < value.length - 4; i++) {
                str += '*';
            }
            return str + value.substring(value.length - 4);
        }
    }
}
</script>
