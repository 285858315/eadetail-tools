<template>
<div>
    <div id='c1'></div>
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
                    switch(field[index]){
                        case "date":
                            // b = new Date(b)
                            break;
                        case "open":
                            b = parseFloat(b)
                            break;
                    }
                    a[field[index]] = b
                    return a
                },{})
            })
        },
        async update(){
            let eurgbp = await this.parseData("history/EURGBP_W1.csv")
            let eurusd = await this.parseData("history/EURUSD_W1.csv")
            let gbpusd = await this.parseData("history/GBPUSD_W1.csv")
            console.log(eurusd[0])
            console.log(gbpusd[0])
            // let ds = new DataSet({
            //     state: {
            //         start: new Date(eurgbp[0].date).getTime(),
            //         end: new Date(eurgbp[10].date).getTime()
            //     }
            // })
            // console.log(ds)

            let data = eurusd.slice(0,1000)
            console.log(data)
            var chart = new G2.Chart({
                container: 'c1',
                forceFit: true,
                height : 400
            });
            chart.source(data);
            chart.interval().position('date*open').color('#CCC')
            chart.render();
        }
    }
}
</script>

<style>

</style>
