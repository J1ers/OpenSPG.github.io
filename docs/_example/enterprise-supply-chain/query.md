---
title: 查询任务
order: 4
---

## 场景1：企业信用评级特征生成

需求：在企业信用评级中，假定需要得到如下决策因子 <br>
1）主供应商关系 <br>
2）企业生产产品所在行业 <br>
3）企业资金近1月、3月、6月转账流水 <br>
4）企业资金近1月、3月、6月流水差 <br>
5）实控人相关公司信息 <br>
但在原有图谱中，只有资金转账、法人代表信息，无法直接获取上述特征，本例演示如果通过SPG完成如上5个特征获取。 <br>
特征定义在schema文件中，可点击查看[企业供应链图谱schema](https://github.com/OpenSPG/openspg/blob/master/python/knext/examples/supplychain/schema/supplychain.schema)。 <br>

**特征1：先定义企业和企业间的主供应链关系，规则定义如下**

```
Define (s:Compnay)-[p:mainSupply]->(o:Company) {
    Structure {
        (s)-[:product]->(upProd:Product)-[:hasSupplyChain]->(downProd:Product)<-[:product]-(o),
        (o)-[f:fundTrans]->(s)
        (otherCompany:Company)-[otherf:fundTrans]->(s)
    }
    Constraint {
        // 计算公司o的转入占比
        otherTransSum("总共转入金额") = group(s).sum(otherf.transAmt)
        targetTransSum("o转入的金额总数") = group(s,o).sum(f.transAmt)
        transRate = targetTransSum*1.0/(otherTransSum + targetTransSum)
        R1("占比必须超过50%"): transRate > 0.5
    }
}
```

**特征2：企业生成产品所在行业**

```
Define (s:Compnay)-[p:belongToIndustry]->(o:Industry) {
    Structure {
        (s)-[:product]->(c:Product)-[:belongToIndustry]->(o)
    }
    Constraint {
    }
}
```

**特征3：企业资金近1月、3月、6月转账流水**

```
// 近1个月流出金额
Define (s:Compnay)-[p:fundTrans1Month]->(o:Int) {
    Structure {
        (s)-[f:fundTrans]->(c:Company)
    }
    Constraint {
        R1("近1个月的流出资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 30
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}

// 近3个月流出金额
Define (s:Compnay)-[p:fundTrans3Month]->(o:Int) {
    Structure {
        (s)-[f:fundTrans]->(c:Company)
    }
    Constraint {
        R1("近4个月的流出资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 90
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}

// 近6个月流出金额
Define (s:Compnay)-[p:fundTrans6Month]->(o:Int) {
    Structure {
        (s)-[f:fundTrans]->(c:Company)
    }
    Constraint {
        R1("近5个月的流出资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 180
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}

// 近1个月流入金额
Define (s:Compnay)-[p:fundTrans1MonthIn]->(o:Int) {
    Structure {
        (s)<-[f:fundTrans]-(c:Company)
    }
    Constraint {
        R1("近1个月的流入资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 30
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}

// 近3个月流入金额
Define (s:Compnay)-[p:fundTrans3MonthIn]->(o:Int) {
    Structure {
        (s)<-[f:fundTrans]-(c:Company)
    }
    Constraint {
        R1("近3个月的流入资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 90
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}


// 近6个月流入金额
Define (s:Compnay)-[p:fundTrans6MonthIn]->(o:Int) {
    Structure {
        (s)<-[f:fundTrans]-(c:Company)
    }
    Constraint {
        R1("近6个月的流入资金"): date_diff(from_unix_time(now(), 'yyyyMMdd'),f.transDate) < 180
        totalOut = group(s).sum(transAmt)
        o = totalOut
    }
}
```

**特征4：企业资金近1月、3月、6月流水差**

```
// 近1个月资金流水差
Define (s:Company)-[p:cashflowDiff1Month]->(o:Integer) {
    Structure {
        (s)
    }
    Constraint {
        // 此处引用特征3中的规则
        fundTrans1Month = rule_value(s.fundTrans1Month == null, 0, s.fundTrans1Month)
        fundTrans1MonthIn = rule_value(s.fundTrans1MonthIn == null, 0, s.fundTrans1MonthIn)
        o = fundTrans1Month - fundTrans1MonthIn
    }
}

// 近3个月资金流水差
Define (s:Company)-[p:cashflowDiff3Month]->(o:Integer) {
    Structure {
        (s)
    }
    Constraint {
        // 此处引用特征3中的规则
        fundTrans3Month = rule_value(s.fundTrans3Month == null, 0, s.fundTrans3Month)
        fundTrans3MonthIn = rule_value(s.fundTrans3MonthIn == null, 0, s.fundTrans3MonthIn)
        o = fundTrans3Month - fundTrans3MonthIn
    }
}

// 近6个月资金流水差
Define (s:Company)-[p:cashflowDiff6Month]->(o:Integer) {
    Structure {
        (s)
    }
    Constraint {
        fundTrans6Month = rule_value(s.fundTrans6Month == null, 0, s.fundTrans6Month)
        fundTrans6MonthIn = rule_value(s.fundTrans6MonthIn == null, 0, s.fundTrans6MonthIn)
        o = fundTrans6Month - fundTrans6MonthIn
  }
}
```

**特征5：同实控人公司**

```
// 定义同法人关系
Define (s:Compnay)-[p:sameLegalReprensentative]->(o:Company) {
    Structure {
        (s)<-[:legalReprensentative]-(u:Person)-[:legalReprensentative]->(o)
    }
    Constraint {
    }
}
```

通过如下GQL执行得到某个公司的具体特征:

```
MATCH
    (s:SupplyChain.Company)
RETURN
    s.id, s.fundTrans1Month, s.fundTrans3Month,
    s.fundTrans6Month, s.fundTrans1MonthIn, s.fundTrans3MonthIn,
    s.fundTrans6MonthIn, s.cashflowDiff1Month, s.cashflowDiff3Month, s.cashflowDiff6Month
```

```
MATCH
    (s:SupplyChain.Company)-[:mainSupply]->(o:SupplyChain.Company)
RETURN
    s.id, o.id
```

```
MATCH
    (s:SupplyChain.Company)-[:belongToIndustry]->(o:SupplyChain.Industry)
RETURN
    s.id, o.id
```

```
MATCH
    (s:SupplyChain.Company)-[:sameLegalRepresentative]->(o:SupplyChain.Company)
RETURN
    s.id, o.id
```

## 场景2：企业供应链发生变化

假设供应链发生如下变化： <br>

```
"钱****份限公司"发布公告，生产产品“三轮摩托车，二轮摩托车”变更为“两轮摩托车”，则"三角**轮胎股份"和钱"****份限公司"的主供应链关系自动断裂，"三角**轮胎股份"和"钱****份限公司"不再具有主供应链关系
```

变更后的数据存在CompanyUpdate.csv:

```cypher
id,name,products
CSF0000001662,浙江**摩托**限公司,"汽车-摩托车制造-二轮摩托车"
```

重新提交任务:

```cypher
knext builder submit CompanyUpdate
```

执行完成后再次查询,只会返回二轮摩托车，而三轮摩托车不再被关联:

```
MATCH
    (s:SupplyChain.Company)-[:product]->(o:SupplyChain.Product)
WHERE
    s.id = "CSF0000001662"
RETURN
    s.id, o.id
```

## 场景3：产业链影响

事件内容如下:

```
id,name,subject,index,trend
1,顺丁橡胶成本上涨,商品化工-橡胶-合成橡胶-顺丁橡胶,价格,上涨
```

提交事件数据:

```
knext builder submit ProductChainEvent
```

传导链路如下: <br>
![image.png](https://mdn.alipayobjects.com/huamei_xgb3qj/afts/img/A*mJWRQJek1BsAAAAAAAAAAAAADtmcAQ/original) <br>
顺丁橡胶成本上升，被分类为产业链价格上涨事件，如下DSL:

```
// ProductChainEvent为一个具体的事件实例，当其属性满足价格上涨条件时，该事件分类为价格上涨事件
Define (e:ProductChainEvent)-[p:belongTo]->(o:`TaxonofProductChainEvent`/`价格上涨`) {
    Structure {
    }
    Constraint {
        R1: e.index == '价格'
        R2: e.trend == '上涨'
    }
}
```

产业链价格上涨，在如下条件下，会导致特定公司的成本上升

```
// 定义了价格上涨和企业成本上升的规则
Define (s:`TaxonofProductChainEvent`/`价格上涨`)-[p:leadTo]->(o:`TaxonofCompanyEvent`/`成本上涨`) {
    Structure {
        //1、找到产业链事件的主体，本例中为顺丁橡胶
        //2、找到顺丁橡胶的下游产品，本例中为斜交轮胎
        //3、找到生成斜交轮胎的所有企业，本例中为三角**轮胎股份
        (s)-[:subject]->[prod:Product]-[:hasSupplyChain]->(down:Product)<-[:product]-(c:Company)
    }
    Constraint {
    }
    Action {
        // 创建一个公司成本上升事件，主体为查询得到的三角**轮胎股份
        downEvent = createNodeInstance(
            type=CompanyEvent,
            value={
                subject=c.id
                trend="上涨"
                index="成本"
            }
        )
        // 由于这个事件是通过产业链价格上涨引起，故在两者之间增加一条边
        createEdgeInstance(
            src=s,
            dst=downEvent,
            type=leadTo,
            value={
            }
        )
    }
}
```

可通过如下查询语句查出某个事件产生的影响

```cypher
MATCH
    (s:SupplyChain.ProductChainEvent)-[:leadTo]->(o:SupplyChain.CompanyEvent)
RETURN
    s.id,s.subject,o.subject,o.name
```
