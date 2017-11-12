---
layout: post
title: "cleanCode"
date: 2017-09-03 23:00:42 +0800
categories： 基础
tags: java
---


这次做一个页面排序的功能，这个页面会展示一个表或几个表联合查询的各个字段，页面提供根据其中的几个字段进行排序的功能，而这样的页面有5个，且这五个页面中需要排序的字段是一样的。  
通过分析，我把这些排序的字段再次分为两类，一类是不可通过数据库字段做排序的，因为这些字段的排序规则由人工划定优先级，类似[a,f,t]这样，并且这些字段区分度不高；另一类是可通过数据库直接做排序查询的，包括varchar类型，Bigdecimal类型等；同时五个页面中对排序的字段是一样的，所以我可以使用相同的排序规则。   
分析之后，我决定构造一个抽象的基类，这个基类对外提供一个接口，通过传入搜索条件和排序的字段，返回符合条件的排序后的数据。  

~~~
public abstract class DataQueryService<T extends AbstractSearchConditon> {  
 protected List<String> sortKeys = Lists.newArrayList("COLUMN1", "COLUMN2", "COLUMN3", "COLUMN4");
    public List queryBoxDataByCondition(T queryCondition) {
        if (sortKeys.contains(queryCondition.getSortKey())) {
            return queryDataByConditionwithoutIndex(queryCondition);
        } else {
            return queryDataByConditionwithIndex(queryCondition);
        }
    }
~~~
 
sortKeys这个列表中包含的时需要做内存排序的排序字段，queryDataByConditionwithoutIndex方法实现查询数据并在内存中排序；queryDataByConditionwithIndex实现数据查询，排序通过数据库排序来实现。由于不同的页面的数据是通过查询不同的数据表来展示的，那么各个不同页面的查询方法通过实现这两个方法来完成。  

~~~
@Service
public class APageDataQueryServiceImpl extends DataQueryService<APageSearchConditon> {
    @Override
    public List queryDataByConditionwithIndex(APageSearchConditon queryCondition) {
        return workspaceDao.queryPersonalBoxDataSorted(queryCondition);
    }
    @Override
    protected List queryDataByConditionwithoutIndex(APageSearchConditon queryCondition) {
    //todo
    }
}
~~~

完成第一个子类APageDataQueryServiceImpl的两个方法之后，继续其他页面时会发现不管是哪个页面，在内存中的排序规则也是一致的，区别只在于从数据库中查询数据的数据表不一样，于是又有了对基类的再次改造。

~~~
public abstract class DataQueryService<T extends AbstractSearchConditon> {
 protected List<String> sortKeys = Lists.newArrayList("COLUMN1", "COLUMN2", "COLUMN3", "COLUMN4");


    public List queryBoxDataByCondition(T queryCondition) {
        logger.debug("queryBoxDataByCondition-begin:{}", queryCondition);
        if (sort1.contains(queryCondition.getSortKey())) {
            return queryDataByConditionwithoutIndex(queryCondition);
        } else {
            return queryDataByConditionwithIndex(queryCondition);
        }
    }

    /**
     * 不根据数据库索引进行排序
     * 这个排序规则很简单，根据queryCondition中的offset，计算出当前要显示的column1的值和对应这个值的offset
     * 如当前的column1有a,b,c,d这四个值，按照[a,c,d,b]进行排序。经过统计之后，值为a的有5条记录，b有3条记录，c有3条记录，d有2条记录.queryCondition中的查询条件为offset=4，pageSize=4，且正序排列；那么对应的展示在页面上的就是展示值为a的最后一条记录，值为c的3条记录。知道了这些之后，就可以直接从数据库中分别查询值为a和c的对应条数的记录拼装后返回即可。
     * @param queryCondition
     * @return
     */
    private List queryDataByConditionwithoutIndex(T queryCondition) {
        List result = Lists.newArrayList();
        Map<String, Map<String, Object>> countMap = countBoxDataByColumnName(queryCondition);
        Integer flag = 0;//标识当前callno所在位置
        Integer initOffset = queryCondition.getOffset();
        Integer initPageSize = queryCondition.getPageSize();
        Integer offset = initOffset;
        Integer leftNum = queryCondition.getPageSize();
        List<String> countColumnList = getSortOrder(sortKey,sortRule, countMap);//获取需求中规定的排序规则
        for (String columnValue : countColumnList) {
            Integer slit = countMap.get(columnValue) == null ? 0 : Integer.valueOf(String.valueOf(countMap.get(columnValue).get("num")));//标识当前callno总数量
            if (flag + slit > offset && leftNum > 0) {
                queryCondition.setOffset(offset - flag);
                queryCondition.setPageSize(leftNum);
                List slitData = queryBoxDataByColumnName(queryCondition, columnValue);
                result.addAll(slitData);
                if (result.size() < initPageSize) {
                    flag += slitData.size();
                    offset += slitData.size();
                    leftNum -= slitData.size();
                } else {
                    return result.subList(0, initPageSize);
                }
            } else {
                flag += slit;
            }
        }
        return result;
    }

    protected abstract List queryDataByConditionwithIndex(T queryCondition);
   /**
     * 统计数据库中，待排序字段column1各个值的统计数据
     *
     * @param queryCondition 查询条件
     * @return 外层Map：key为column1的值value1，value为统计数据的Map；内层Map：key为sql查询时的别名，如key为count，value为column1为value1的总数
     */
    protected abstract Map<String, Map<String, Object>> countBoxDataByColumnName(T queryCondition);

    /**
     * 根据column1指定的value（columnValue）查询数据
     *
     * @param queryCondition 查询条件，包含offset和pageSize
     * @param columnValue    column1的值
     * @return 查的的数据
     */
    protected abstract List queryBoxDataByColumnName(T queryCondition, String columnValue);

}

~~~

通过这样一个抽象之后，各个页面的查询子类只需要实现基类中的三个方法即可完成按照指定字段进行查询数据并排序的功能。  
另外是写代码的过程中，发现的现有代码的问题。现有的项目架构没有一个统一的规划，各个页面的操作其实有很多共通的地方，例如这次为了添加排序规则，我把各个页面的查询类QueryCondition的公共参数抽象到了一个基类中，从而实现了数据查询功能的代码抽象和复用，一方面减少了代码量，另一方面扩展性更好，后续也有利于开发维护。    

>初接触代码，了解到代码质量这个概念的时候，我并不清楚什么样的代码是好代码。幸运的是我很听话地遵循了《clean code》的教导。如今在面对一些代码的时候，我依旧会疑惑这样的代码是好代码吗？我应该怎样写会更好一点？代码review的时候，不同的程序员对这个问题有不同的答案，连我自己也会随着编程经验的丰富而改变自己的编程习惯。但是不管怎样，我始终坚信且不受其他人影响的准则是：可读性好+简洁+可扩展  
>好的代码会让bug无处可藏,当一个系统的代码变得让人不敢动的时候，这个系统已经成为了一个烂系统。
