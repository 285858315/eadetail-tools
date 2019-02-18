<template>
<div>
    index...
</div>
</template>

<script>
import G2 from '@antv/g2'
import DataSet from '@antv/data-set'

export default {
    data(){
        return {
            base:"https://raw.githubusercontent.com/285858315/eadetail-tools/master/data/"
        }
        
    },
    mounted(){
        this.update()
    },
    methods:{
        async getData(file){
            let res = await fetch(`${this.base}${file}`)
            return res.text()
        },
        async parseData(file){
            let res = await this.getData(file)
            res = res.split("\n")
            let field = res.shift().toLocaleLowerCase().replace(/\<|\>/g,'').split("	")
            return res.map(item => item.split("	")).map(item => {
                return item.reduce((a,b,index) => {
                    a = a || {}
                    a[field[index]] = b
                    return a
                },{})
            })
        },
        async update(){
            let eurgbp = await this.parseData("history/EURGBP_D1.csv")
            let eurusd = await this.parseData("history/EURUSD_D1.csv")
            let gbpusd = await this.parseData("history/GBPUSD_D1.csv")
            console.log(eurusd[0])
            console.log(gbpusd[0])
            let ds = new DataSet({
                state: {
                    start: new Date(eurgbp[0].date).getTime(),
                    end: new Date(eurgbp[10].date).getTime()
                }
            })
            console.log(ds)
        }
    }
}
</script>

<style>

</style>
