**apply中的WindowFunction使用方法**
```
.keyby(0,1)
.timeWindow(Time.seconds(60))
                            输入tuple4                                    输出tuple6
.apply(new WindowFunction<Tuple4<String, String, Double, Long>, Tuple6<String, String, Double, Double, 
 Tuple里面有keyby的数据 TimeWindow可以get出窗口开始和结束时间
Double, Long>, Tuple, TimeWindow>() {
                    @Override
                    具体的逻辑                                       此处是keyby进行分组后的一组数据
                    public void apply(Tuple tuple, TimeWindow window, Iterable<Tuple4<String, String, Double, Long>> input, Collector<Tuple6<String, String, Double, Double, Double, Long>> out) throws Exception {
                        将数据迭代出来进行逻辑处理
                        Iterator<Tuple4<String, String, Double, Long>> it = input.iterator();
                        List<Tuple4<String, String, Double, Long>> dataList = new ArrayList<>();

                        Long count = 0L;
                        Double sum = 0.0;
                        while (it.hasNext()) {
                            Tuple4<String, String, Double, Long> next = it.next();
                            sum += next.f2;
                            count++;
                            dataList.add(next);
                        }
                        Collections.sort(dataList, new Comparator<Tuple4<String, String, Double, Long>>() {
                            @Override
                            public int compare(Tuple4<String, String, Double, Long> o1, Tuple4<String, String, Double, Long> o2) {
                                return o2.f2.compareTo(o1.f2);
                            }
                        });

                        double avg = sum / count;
                        double max = dataList.get(0).f2;
                        double min = dataList.get(dataList.size() - 1).f2;

                        String devId = tuple.getField(0);
                        String metric = tuple.getField(1);
                         将数据发射输出
                        out.collect(Tuple6.of(devId, metric, max, min, avg, window.getStart()));

                    }
                });