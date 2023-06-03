---
layout: post
title: 敏感词汇过滤
categories: word
tags: DFA
date: 2023-06-03
---
敏感词过滤的算法，使用 DFA(Deterministic Finite Automaton) 算法，确定有穷自动机。其特征为：有一个有限状态集合和一些从一个状态通向另一个状态的边，每条边上标记有一个符号，其中一个状态是初态，某些状态是终态。但不同于不确定的有限自动机，DFA中不会有从同一状态出发的两条边标志有相同的符号。
<!--more-->
## 1.存储结构
原本准备使用 Redis 来存储，但是感觉性能不如直接存在 HashMap 中好，因为一般敏感词使用的频率还是比较高的。为了保证访问速度，直接用 HashMap 来存储，不用 ConcurrentHashMap，在更新敏感词库时，参考 CopyOnWriteArrayList，可以直接用新生成的敏感词树替换原有的树。

## 2.Code
因为 Java Character 使用两个字节表示，所以有些字符需要两个 Char 表示，但是并不影响敏感词的过滤
```java
@Component
public class DFAFilterSensitiveWord {
    private Map<String, Object> tree = new HashMap<>();

    @Resource
    private SensitiveWordMapper mapper;

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class R {
        private Integer code;
        private String msg;
        //保证 json key 的顺序
        private Map<String, Object> data = new TreeMap<>();

        public static R success() {
            return new R(200, "success", new TreeMap<>());
        }

        public static R error() {
            return new R(400, "failure", null);
        }

        public R data(String key, Object value) {
            data.put(key, value);
            return this;
        }
    }

    @PostConstruct
    private void readSensitiveWord() {
        final List<SensitiveWord> words = mapper.getWords();
        final Set<String> stringSet = words.stream().map(SensitiveWord::getWord).collect(Collectors.toSet());
        Map<String, Object> origin = tree;
        tree = generateSensitiveTree(stringSet);
        origin.clear();
    }

    public synchronized void updateSensitiveTree() {
        readSensitiveWord();
    }

    private Map<String, Object> generateSensitiveTree(Set<String> words) {
        Map<String, Object> sensitiveTree = new HashMap<>();
        for (String word : words) {
            Map<String, Object> current = sensitiveTree;
            for (int i = 0; i < word.length(); i++) {
                String key = String.valueOf(word.charAt(i));

                if (current.containsKey(key)) {
                    current = (Map<String, Object>) current.get(key);
                } else {
                    Map<String, Object> map = new HashMap<>();
                    current.put(key, map);
                    current = map;
                }

                if (i == word.length() - 1) {
                    current.put("isEnd", true);
                }
            }
        }

        return sensitiveTree;
    }

    public R filterSensitiveWord(String content) {
        Map<String, Object> current = tree;
        StringBuilder word = new StringBuilder();
        StringBuilder resStr = new StringBuilder(content);
        List<String> sensitiveWords = new ArrayList<>();

        boolean matching = false;
        int index = 0;
        while (index < content.length()) {
            String c = String.valueOf(content.charAt(index));
            if (current.containsKey(c)) {
                matching = true;
                current = (Map<String, Object>) current.get(c);
                word.append(c);
                index++;
            } else {
                if (matching) {
                    if ((boolean) current.getOrDefault("isEnd", false)) {
                        resStr.replace(index - word.length(), index, "*".repeat(word.length()));
                        sensitiveWords.add(word.toString());
                    }
                    current = tree;
                    word = new StringBuilder();
                    matching = false;
                } else {
                    index++;
                }
            }
        }

        //最后一个词是敏感词
        if (matching) {
            if ((boolean) current.getOrDefault("isEnd", false)) {
                resStr.replace(index - word.length(), index, "*".repeat(word.length()));
                sensitiveWords.add(word.toString());
            }
        }

        return R.success().data("content", resStr).data("sensitiveWords", sensitiveWords);
    }
}
```

## 3.测试
```java
@RestController
@RequestMapping("/sensitive")
public class SensitiveWordController {

    @Autowired
    private DFAFilterSensitiveWord filterSensitiveWord;

    @PostMapping("/filter")
    public R filter(@RequestBody String content) {
        return filterSensitiveWord.filterSensitiveWord(content);
    }
}
```

敏感词
```
法轮功
falungong
taidu
1450
微信
🐷
SB
🔪
```

RequestBody
```
到南京时，有朋友约去游逛，勾留了一日；第二日上午便须渡江到浦口，下午上车北去。父亲因为事忙，本已说定不送我，
叫旅馆里一个熟识的茶房陪我同去。他再三嘱咐茶房，甚是仔细。但他终于不放心，怕茶房不妥帖；颇踌躇了一会。
其🐷实我那年已二十岁，北京已来往过两三次，是没有什么要紧的了。他踌躇了一会，终于决定还是自己送我去。
我再三劝他不必去；他只说：“不要紧，他们去不好！”
我们过了江，进了taidu车站。我SB买票，他忙着照看行李。行李太多了，得向脚夫行些小费才可过去。
他便又忙着和他们讲价钱。我那时真是聪明过分，总觉他说话不大漂亮，非自己插嘴不可，但他终于讲falungong定了价钱；
就送我上车。他给我拣定了靠车门的一张椅子；我将他给我做的紫毛大衣铺好座位。他嘱我路上小心，夜里要警醒些，不要受凉。
又嘱托茶房好好照应我。我心里法轮功暗笑他的迂；他们只认得钱，托他们1450只是白托！而且我这样大年纪的人，难道还不能料理自🔪己么？
唉，我现在想想，那时真是太聪明了！
```

Response
```json
{
    "code": 200,
    "msg": "success",
    "data": {
        "content": "到南京时，有朋友约去游逛，勾留了一日；第二日上午便须渡江到浦口，下午上车北去。父亲因为事忙，本已说定不送我，\n叫旅馆里一个熟识的茶房陪我同去。他再三嘱咐茶房，甚是仔细。但他终于不放心，怕茶房不妥帖；颇踌躇了一会。\n其**实我那年已二十岁，北京已来往过两三次，是没有什么要紧的了。他踌躇了一会，终于决定还是自己送我去。\n我再三劝他不必去；他只说：“不要紧，他们去不好！”\n我们过了江，进了*****车站。我**买票，他忙着照看行李。行李太多了，得向脚夫行些小费才可过去。\n他便又忙着和他们讲价钱。我那时真是聪明过分，总觉他说话不大漂亮，非自己插嘴不可，但他终于讲*********定了价钱；\n就送我上车。他给我拣定了靠车门的一张椅子；我将他给我做的紫毛大衣铺好座位。他嘱我路上小心，夜里要警醒些，不要受凉。\n又嘱托茶房好好照应我。我心里***暗笑他的迂；他们只认得钱，托他们****只是白托！而且我这样大年纪的人，难道还不能料理自**己么？\n唉，我现在想想，那时真是太聪明了！",
        "sensitiveWords": [
            "🐷",
            "taidu",
            "SB",
            "falungong",
            "法轮功",
            "1450",
            "🔪"
        ]
    }
}
```