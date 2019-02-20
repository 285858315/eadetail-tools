<template>
<div>
</div>
</template>

<script>

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
                            // b = b.replace(/\./g,'')
                            break;
                        case "open":
                            b = parseFloat(b)
                            break;
                    }
                    a[field[index]] = b
                    return a
                },{})
            }).filter(item => item.date)
        },
        log(){
            console.log.apply(null,arguments)
        },
        toFixed(number,digits = 2){
            return parseFloat(number.toFixed(digits))
        },
        async update(){
            const history = {
                eurgbp:await this.parseData("history/EURGBP_H1.csv"),
                eurusd:await this.parseData("history/EURUSD_H1.csv"),
                gbpusd:await this.parseData("history/GBPUSD_H1.csv")
            }
            history.eurgbp = history.eurgbp.slice(0,history.eurgbp.length - 100)
            let group_id = 0
            let groups = []
            let 最大分组数 = 0, 最大分组时间, 最大回撤 = 0, 最大回撤时间
            const Symbol = {
                point:0.00001,
            }
            /**
             * 添加分组
             */
            const newGroup = (opts) => {
                group_id += 1
                this.log(`${group_id}:添加`)
                groups.push({
                    max_loss:-300,
                    point:0,
                    real_point:0,
                    id:group_id,
                    open:opts.datetime,
                    // lots:(getOpenGroups().length + 1)* 0.01,
                    lots: 0.01,
                    act:opts.act,
                })
            }

            const getOpenGroups = () => {
                return groups.filter(item => !item.is_close)
            }


            const Acts = [
                {eurgbp:"buy",eurusd:"sell",gbpusd:"buy"},
                {eurgbp:"sell",eurusd:"buy",gbpusd:"sell"}
            ]
            const d5 = 5 * 24 * 60 * 60 * 1000
            const d10 = 10 * 24 * 60 * 60 * 1000
            const d1 =  24 * 60 * 60 * 1000
            /**
             * 验证分组
             */
            const checkGroups = (tick,index) => {
                const datetime = `${tick.date+(tick.time  &&  " " + tick.time  || "")}`
                let open_groups = getOpenGroups()
                if(open_groups.length == 0){
                    newGroup({datetime,act:0})
                }
                open_groups = getOpenGroups()
                const open0_groups = open_groups.filter(item => item.act == 0)
                const open1_groups = open_groups.filter(item => item.act == 1)
                let _最大回撤 = 0
                open_groups.forEach(item => {
                    let acts = Acts[item.act]
                    let point = 0
                    for(let symbol in acts){
                        switch(acts[symbol]){
                            case "buy":
                                point += this.toFixed((history[symbol][index].close - history[symbol][index].open) / Symbol.point)
                                break;
                            case "sell":
                                point += this.toFixed((history[symbol][index].open - history[symbol][index].close) / Symbol.point)
                                break;
                        }
                    }
                    item.point += point
                    item.real_point = item.point * this.toFixed(item.lots / 0.01)
                    _最大回撤 += item.real_point
                    if(item.point > 500 || (new Date(datetime).getTime() - new Date(item.open).getTime() > d10)){
                        item.is_close = true
                        item.close = datetime
                        item.days = (new Date(datetime).getTime() - new Date(item.open).getTime()) / d1
                        this.log(`${item.id}:平仓%c 时间:${datetime}%c 利润:${item.real_point} `,'color:#999','color:blue;')
                    }else{
                        //更新最大回测
                        if(item.real_point < item.max_loss){
                            item.max_loss = item.real_point
                            // this.log(`${item.id}:${item.real_point} %c 时间:${datetime}`,'color:#999')
                        }
                    }
                })
                // if(last && Math.abs(last.point) > 1000){
                let last = open_groups[open_groups.length - 1]
                if(last && Math.abs(last.point) > 200){
                    // newGroup({datetime,act:open0_groups.length > open1_groups.length ? 1 : 0})
                    newGroup({datetime,act:groups.length % 2})
                }
                if(open_groups.length > 最大分组数){
                    最大分组数 =  open_groups.length 
                    最大分组时间 = datetime
                }
                if(_最大回撤 < 最大回撤){
                    最大回撤时间 = datetime
                    最大回撤 = _最大回撤
                }
            }

            
            history.eurgbp.forEach((item,index) => {
                checkGroups(item,index)
            })

            const close_stat = groups.filter(item => item.is_close).reduce((res,item,index) => {
                res = res || {}
                res.count = res.count || 0
                res.count += 1
                res.point = res.point || 0
                res.point += item.point
                res.real_point = res.real_point || 0
                res.real_point += item.real_point
                res.days = res.days || 0
                res.days += item.days || 0
                res.groups = res.groups || []
                res.groups.push(item)
                return res
            },{})
            const open_stat = groups.filter(item => !item.is_close).reduce((res,item,index) => {
                res = res || {}
                res.count = res.count || 0
                res.count += 1
                res.point = res.point || 0
                res.point += item.point
                res.real_point = res.real_point || 0
                res.real_point += item.real_point
                res.groups = res.groups || []
                res.groups.push(item)
                return res
            },{})
            this.log(`共${groups.length}组`,{最大分组数,最大分组时间,最大回撤,最大回撤时间})
            this.log("未平仓",open_stat)
            this.log("已平仓",close_stat)
        }
    }
}
</script>

<style>

</style>
