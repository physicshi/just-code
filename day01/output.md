```js
//实现以下输出
    let inArr = [
        {
            company: 'COM1',
            customer: 'CUS1'
        },
        {
            company: 'COM1',
            customer: 'CUS2',
        },
        {
            company: 'COM2',
            customer: 'CUS3'
        }
    ]
    let outArr = [
      {
          company: 'COM1',
          custome: ['CUS1', 'CUS2']
      },
      {
          company: 'COM2',
          custome: ['CUS3']
      }
    ] 
```

代码：

```js
 
    function getNewArrFn(inArr) {
        let outArr=[];//输出的值
        for (let val of inArr) {
            //不是空数组就判断输出的数组中每个元素的company是不是和inArr中正在遍历的值相等 
            if (outArr.length > 0) {
                outArr.forEach((item, index) => {
                    if (val.company == item.company) {
                        //相等就追加到custom属性中
                        item.custome.push(val.customer)
                    } else {
                        //不相等就在输出数组中追加数组
                        outArr.push({
                            company: val.company,
                            custome: [val.customer],
                        })
                    }
                })
            }
            else {
                //空数组就追加
                outArr.push({
                    company: val.company,
                    custome: [val.customer],
                })
            }
        }
        return outArr
    }
 
    console.log(JSON.stringify(getNewArrFn(inArr),null,2))
```

