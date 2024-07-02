**process内使用KeyedProcessFunction**
liststate可以设置检查点当程序在某时刻停止再启动会继续(记录偏移量)
```
keyBy("windowEnd").process(new KeyedProcessFunction<Tuple, ItemCount, String>() {

            ListState<ItemCount> listState = null;



            //3.定时器实现逻辑
            @Override
            public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {

                ArrayList<ItemCount> itemCounts = new ArrayList<>();
                //将listState内的数据取出
                for (ItemCount itemCount : listState.get()) {

                    itemCounts.add(itemCount);

                }

                //排序
                Collections.sort(itemCounts, new Comparator<ItemCount>() {
                    @Override
                    public int compare(ItemCount o1, ItemCount o2) {
                        return o1.count.compareTo(o2.count);
                    }
                });

                StringBuffer stringBuffer = new StringBuffer("时间 :"+sdt.format(itemCounts.get(0).windowEnd));

                for (int i = 0;i<itemCounts.size();i++){

                    ItemCount itemCount = itemCounts.get(i);

                    stringBuffer.append("商品ID :"+itemCount.itemID+" 点击量 :"+itemCount.count+"\n");


                }
                //发送出去
                out.collect(stringBuffer.toString());
                //清理list
                itemCounts.clear();



            }

            @Override
            //1.将ListStateDescriptor描述创建出来作为全局使用
            public void open(Configuration parameters) throws Exception {

                ListStateDescriptor<ItemCount> jk = new ListStateDescriptor<>(
                        "jk",//名称
                        TypeInformation.of(new TypeHint<ItemCount>() {
                        }) //类型


                );

                //创建出listState
                listState = getRuntimeContext().getListState(jk);


            }

            @Override
            //2.将数据添加到listState
            public void processElement(ItemCount value, Context ctx, Collector<String> out) throws Exception {


                listState.add(value);
                //创建定时器
                ctx.timerService().registerEventTimeTimer(value.windowEnd + 1);

            }

        });