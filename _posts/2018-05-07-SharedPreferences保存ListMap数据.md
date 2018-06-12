---
layout:     post                    # 使用的布局（不需要改）
title:      SharedPreferences保存ListMap数据     # 标题 
subtitle:   List<Map<String, Object>>数据保存    #副标题
date:       2018-05-07               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-github-cup.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - 数据
    
---


# SharedPreferences保存List<Map<String, Object>>数据

---

众所周知SharedPreferences是以key-vlues存储的xml文件格式，业务需求需要将List<Map<String, Object>>数据格式的数据保存下来，以便下次读取显示，如果用sqlite来保存的话有点杀鸡用牛刀的感觉，毕竟数据量并不大。

下面记录一下如何用SharedPreferences保存List<Map<String, Object>>数据

## 解决方案：

先将数据转化成JSON,然后保存字符串，取出时再将JSON转化成数组

## 保存数据

    public void saveInfo(Context context, String key, List<Map<String, Object>> datas) {
        JSONArray mJsonArray = new JSONArray();
        for (int i = 0; i < datas.size(); i++) {
            Map<String, Object> itemMap = datas.get(i);
            Iterator<Map.Entry<String, Object>> iterator = itemMap.entrySet().iterator();

            JSONObject object = new JSONObject();

            while (iterator.hasNext()) {
                Map.Entry<String, Object> entry = iterator.next();
                try {
                    object.put(entry.getKey(), entry.getValue());
                } catch (JSONException e) {

                }
            }
            mJsonArray.put(object);
        }

        SharedPreferences sp = context.getSharedPreferences("err_port", Context.MODE_PRIVATE);
        SharedPreferences.Editor editor = sp.edit();
        editor.putString(key, mJsonArray.toString());
        editor.commit();
    }


## 读取数据

    public List<Map<String, Object>> getInfo(Context context, String key) {
        List<Map<String, Object>> datas = new ArrayList<Map<String, Object>>();
        SharedPreferences sp = context.getSharedPreferences("err_port", Context.MODE_PRIVATE);
        String result = sp.getString(key, "");
        try {
            JSONArray array = new JSONArray(result);
            for (int i = 0; i < array.length(); i++) {
                JSONObject itemObject = array.getJSONObject(i);
                Map<String, Object> itemMap = new HashMap<String, Object>();
                JSONArray names = itemObject.names();
                if (names != null) {
                    for (int j = 0; j < names.length(); j++) {
                        String name = names.getString(j);
                        String value = itemObject.getString(name);
                        itemMap.put(name, value);
                    }
                }
                datas.add(itemMap);
            }
        } catch (JSONException e) {

        }

        return datas;
    }