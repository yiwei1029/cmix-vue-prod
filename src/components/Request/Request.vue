<template>
    <section class="Request">
        <el-row :gutter="10">
            <!-- 左边 -->
            <el-col :span="12">
                <!-- input -->

                <el-card>
                    <el-row><span>Input</span></el-row>
                    <el-row v-for="input  in InputList" :gutter="20" :key="input.id">
                        <el-col :span="6">
                            <el-popover disabled>
                                <el-button slot="reference" type="text">{{ input.amount }}</el-button>
                            </el-popover>
                        </el-col>
                    </el-row>
                    <el-row>
                        <el-col :span="15">
                            <el-button :disabled="disableAdd" style="width: 100%;" @click="addInput">+ add a input
                            </el-button>
                        </el-col>
                    </el-row>
                </el-card>
                <el-card>

                    <!-- form to sumbit c -->
                    <div>
                        <el-form :model="form">
                            <div>Coordination</div>
                            <el-form-item>
                                <el-select style="width: 100%;" v-model="form.CoordToSelect"
                                    placeholder="Select A Coordination">
                                    <el-option v-for="item in Coords" :key="item" :value="item">
                                    </el-option>
                                </el-select>
                            </el-form-item>
                            <div>The c of decomposition</div>
                            <el-form-item>
                                <el-input style="width: 100%; margin-bottom: 10px" placeholder="Please input c"
                                    v-model="form.valueOfC"></el-input>
                                <el-button style="width: 50%;" type="primary" @click="onSubmit">Send
                                    Request</el-button>
                            </el-form-item>
                        </el-form>
                    </div>
                    <br>
                    <!-- 展示decompose细节 -->
                    <div style="display: flex; ">
                        <el-button type="info" style="flex:1;" @click="dialogVisible = true">Check
                            Decompose</el-button>

                        <el-dialog title="Details" :visible.sync="dialogVisible" width="30%">
                            <div v-for="(value, key, index ) in decompose.decomposeData" :key=key>
                                <span style="font-weight: bolder; font-size: large">{{ key }}: </span><span
                                    style="font-weight: lighter" v-for="v in value">[{{ v[0] }}BTC*{{ v[1] }}] </span>
                            </div>
                            <span slot="footer" class="dialog-footer">
                                <el-button @click="dialogVisible = false">cancel</el-button>
                                <el-button type="primary" @click="dialogVisible = false">ok</el-button>
                            </span>
                            <br>
                            <div style="font-family: Georgia">FYI: <i>f</i> is sender and <i>c</i> is coordinator</div>
                        </el-dialog>
                        <el-button type="primary" :disabled="sendTransactionDisable" style="flex:1; "
                            @click="sendTransaction">Send Transaction</el-button>
                    </div>



                </el-card>
            </el-col>
            <!-- 右边 -->
            <el-col :span="12">

            </el-col>
        </el-row>
    </section>
</template>

<script>
import axios from 'axios';
// import eventBus from '../event-bus.js'
// import { BASE_URL } from '../config/index.js'
import BASE_URL from '../config'
export default {
    name: 'Request',
    data() {
        return {
            base_url: BASE_URL,
            // username: this.$store.state.username,
            // disableAdd: false,
            dialogVisible: false,
            sendTransactionDisable: true,
            InputList: [
                // { id: 0, amount: null },

            ],
            OutputList: [
                // { hash: 'bc1quqfl65xqtkprhrygpdpeg4r7q20zyaq8xvxd3a', amount: 0.00571223 },


            ],
            show: true,
            NumberDisplay: false,
            PromptInputHash: '',
            PromptOutputHash: '',
            Coords: ['dg', 'dr', 'boggart'],
            form: {
                // amount: InputList[0].amount,
                CoordToSelect: '',
                valueOfC: 0.3
            },
            receiveDecomposition: false,
            decompose: {},
            // rawDecompose: {}

        }
    },
    components: {},
    watch: {
        // InputList:
        //     (newVal, oldVal) => {
        //         console.log(newVal.length)
        //         if (newVal.length >= 1) {
        //             disableAdd = true
        //         }
        //     }
        username(oldVal) {
            if (oldVal === '') {
                this.$router.push('/login');
            }
        }
    },
    computed: {
        disableAdd() {
            // console.log(this.InputList.length)
            return this.InputList.length > 0
        },
        username() {
            return this.$store.state.username
        },
        commissionRate() {
            return this.$store.state.commissionRate
        }
    },
    mounted() {
        if (this.username === '') {
            this.$router.push('/login')
        }
        console.log(this.commissionRate.toFixed(2))
    },
    methods: {
        sumArr(arr) {
            let sum = 1.0
            for (let i = 0; i < arr.length; i++) {
                sum = sum * parseFloat(arr[i])
            }
            return sum
        },
        onSubmit() {
            let formData = {
                "amount": this.InputList[0].amount,
                "method": this.form.CoordToSelect,
                "c": this.form.valueOfC
            }
            axios.post(`${this.base_url}/${this.username}/pre_transfer`, formData).then(
                resp => {
                    // this.receiveDecomposition = !this.receiveDecomposition
                    // this.decomposeData = resp.data.data
                    this.rawDecompose = { ...resp.data.data }
                    let decomposeData = resp.data.data['processed_data']
                    // console.log(decomposeData)
                    //判断支付coord的费用是否大于budget
                    let sum = 0
                    for (let key in decomposeData) {
                        if (key.startsWith('c')) {
                            for (let item of decomposeData[key]) {
                                // console.log(item)
                                let temp = this.sumArr(item)
                                sum += temp
                            }
                        }
                    }
                    // console.log('sum: ', sum)
                    if (sum > this.$store.state.budget) {
                        this.$message({
                            message: `Insufficient balance! At least ${sum}`,
                            type: 'error'
                        })
                        return
                    }

                    //budget充足
                    let coords = resp.data.data['data']
                    let cValue = resp.data.data['c']
                    let replenishData = resp.data.data['replenished_data']
                    this.decompose = { decomposeData, coords, cValue, replenishData }
                    this.dialogVisible = !this.dialogVisible
                    this.sendTransactionDisable = false
                    // console.log(resp.data.data)
                }
            ).catch(err => { console.log(err) })
        },
        sendTransaction() {
            const loadingInstance = this.$loading({
                lock: true,
                text: 'Transfer in progress please wait a while...',
                spinner: 'el-icon-loading',
                background: 'rgba(0, 0, 0, 0.7)'
            })
            let formData = { ...this.decompose.decomposeData }
            // eventBus.$emit('block_data', 'hello')
            // console.log(formData)
            axios.post(`${this.base_url}/${this.username}/transfer?fee_rate=${this.commissionRate.toFixed(2)}`, formData)
                .then(resp => {
                    // console.log(resp.data)
                    // eventBus.$emit('block_data', resp.data)
                    this.$store.commit('updateBalance', this.InputList[0].amount)
                    this.$store.commit('updateBlockData', resp.data)
                    this.$message
                        ({
                            message: 'transfer successfully',
                            type: 'success',
                            offset: 100,
                            duration: 1000
                        })
                    loadingInstance.close()
                })
                .catch(err => console.log(err))
        },
        addInput() {
            this.$msgbox.prompt('', 'Please input transfer amount', {
                confirmButtonText: 'Confirm',
                cancelButtonText: 'Cancel',
                inputValidator: value => {
                    // string
                    let parseValue = parseFloat(value)
                    if (isNaN(parseValue) || parseValue < 0.1) {
                        return false
                    }
                },
                inputErrorMessage: 'input amount is not legal or too small(>0.1)'
            }).then(({ value }) => {
                // const newAdd = { hash: value, amount:  }
                this.InputList.push({
                    id: this.InputList.length,
                    amount: value
                })
                this.$message({
                    type: 'success',
                    message: 'your input: ' + value,
                    offset: 100,
                    duration: 1000
                });
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: 'Cancel',
                    offset: 100,
                    duration: 1000
                });
            });
        },
        addOutput() {
            this.$msgbox.prompt('please input', 'Output', {
                confirmButtonText: 'Confirm',
                cancelButtonText: 'Cancel',
            }).then(({ value }) => {
                const newAdd = { hash: value, amount: this.InputList[this.InputList.length - 1].amount }
                this.OutputList.push(newAdd)
                this.$message({
                    type: 'success',
                    message: 'your input: ' + value
                });
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: 'Cancel'
                });
            });
        }
    }
}
</script>

<style scoped lang="less">
.el-col {
    overflow: hidden;
    text-overflow: ellipsis;
}

.el-card {
    margin-bottom: 20px;
}

.left-right {
    display: flex;
    justify-content: space-between;
    margin-bottom: 10px;
    align-items: center;

    :first-child {
        flex: 1;
    }

    :last-child {
        flex: 1;
    }

}

// .el-input {
//     // width: 50%;
// }

.number-display {
    display: flex;
    justify-content: space-between;
    margin-bottom: 10px;

    >div {
        background-color: #8977fd;
        color: #fff;
        padding: 4px 6px;
        border-radius: 6px;

    }
}
</style>
