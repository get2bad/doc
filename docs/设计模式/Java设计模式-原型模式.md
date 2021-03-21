# 原型模式

> 原型模式主要解决的问题就是创建᯿复对象，⽽这部分 对象 内容本身⽐较复杂，⽣成过程可能从库或者RPC接⼝中获取数据的耗时较⻓，因此采⽤克隆的⽅式节省时间。

其实这种场景经常出现在我们的身边，只不过很少⽤到⾃⼰的开发中，就像；

1. 你经常 Ctrl+C 、 Ctrl+V ，复制粘贴代码。

2. Java多数类中提供的API⽅法； Object clone() 。

3. 细胞的有丝分裂。



本节，我们来列举一个考试的问题，如果是上机考试，我们见过的大部分的上机考试题是乱序的，每个人的试题总体是一致的，但是顺序是不一致的，这样可以减少作弊的可能性



我们先假设我们的考试内容只有 ```选择题```、```填空题```

+ ChoiceQuestion

```java
@Data
public class ChoiceQuestion {

    // 题目
    private String name;
    // 选项
    private Map<String,String> option;
    // 答案
    private String key;
}
```

+ FeildQuestion

```java
@Data
public class FieldQuestion {

    // 题目
    private String name;
    // 作答的答案
    private String res;
}
```



## 没有应用原型模式

如果我们没有应用原型设计模式，我们会这么写代码：

+ SimpleSolution

```java
public class SimpleSolution {

    /**
     *
     * @param number 学号
     * @param username 学生用户名
     * @return
     */
    public String examination(String number,String username){
        List<ChoiceQuestion> choiceQuestions = new LinkedList<>();
        List<FieldQuestion> fieldQuestions = new LinkedList<>();

        // 本来是数据库读取的内容，我们没有应用数据库，所以直接手动创建
        Map<String,String> q1 = new HashMap<>();
        q1.put("A","1");
        q1.put("B","2");
        q1.put("C","3");
        q1.put("D","4");

        Map<String,String> q2 = new HashMap<>();
        q2.put("A","STRING");
        q2.put("B","INTEGER");
        q2.put("C","CHAR");
        q2.put("D","LIST");

        Map<String,String> q3 = new HashMap<>();
        q3.put("A","Thread");
        q3.put("B","Excutors");
        q3.put("C","ThreadPool");
        q3.put("D","ThreadLocal");

        Map<String,String> q4 = new HashMap<>();
        q4.put("A","Binary");
        q4.put("B","Hex");
        q4.put("C","Hash");
        q4.put("D","VERBOSE");

        choiceQuestions.add(new ChoiceQuestion("1+1=?",q1));
        choiceQuestions.add(new ChoiceQuestion("字符串的数据类型是？",q2));
        choiceQuestions.add(new ChoiceQuestion("线程公用的数据存放对象是？",q3));
        choiceQuestions.add(new ChoiceQuestion("Java 反编译class文件，我们要使用哪个参数？",q4));

        fieldQuestions.add(new FieldQuestion("埋在地下一千年的酒，出来以后叫什么?","酒精"));
        fieldQuestions.add(new FieldQuestion("有一只猪，它走啊走啊，走到了英国,结果他变成了什么?","pig"));
        fieldQuestions.add(new FieldQuestion("上课老师抽查背课文,小猪,小狗,小猫都举手了,老师会叫谁?","小狗，因为旺旺仙贝"));
        fieldQuestions.add(new FieldQuestion("蝴蝶, 蚂蚁, 蜘蛛, 蜈蚣,他们一起工作,最后哪一个没有领到酬劳?","蜈蚣，因为无功不受禄"));

        StringBuilder sb = new StringBuilder();

        sb.append("学生：" + username);
        sb.append("\r\n学号：" + number);
        sb.append("\r\n选择题：：");
        choiceQuestions.forEach(data -> {
            sb.append("\r\n" + data.getName());
            for (String s : data.getOption().keySet()) {
                sb.append("\r\n" + s + ":" + data.getOption().get(s));
            }
        });

        sb.append("填空题：");
        for (FieldQuestion question : fieldQuestions) {
            sb.append("\r\n" + question.getName());
        }

        return sb.toString();
    }
}
```

测试类：

+ Test

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test(){
        SimpleSolution simpleSolution = new SimpleSolution();
        log.info("\r\n" + simpleSolution.examination("123456","隔壁的老王"));
    }
}
```

输出内容：

```console
11:56:05.755 [main] INFO  com.wills.java.prototype.simple.Test - 
学生：隔壁的老王
学号：123456
选择题：：
1+1=?
A:1
B:2
C:3
D:4
字符串的数据类型是？
A:STRING
B:INTEGER
C:CHAR
D:LIST
线程公用的数据存放对象是？
A:Thread
B:Excutors
C:ThreadPool
D:ThreadLocal
Java 反编译class文件，我们要使用哪个参数？
A:Binary
B:Hex
C:Hash
D:VERBOSE填空题：
埋在地下一千年的酒，出来以后叫什么?
有一只猪，它走啊走啊，走到了英国,结果他变成了什么?
上课老师抽查背课文,小猪,小狗,小猫都举手了,老师会叫谁?
蝴蝶, 蚂蚁, 蜘蛛, 蜈蚣,他们一起工作,最后哪一个没有领到酬劳?
```

+ 这样的代码往往都⾮常易于理解，要什么程序就给什么代码，不⾯向对象，只⾯向过程。不考虑扩展性，能⽤就⾏。

+ 但以上的代码有⼀个没有实现的地⽅就是不能乱序，所有⼈的试卷顺序都是⼀样的。如果需要加乱序也是可以的，但复杂度⼜会增加。这⾥不展示具体过多实现，只为后⽂对⽐᯿构。

+ ⽽且以上这样的代码⾮常难扩展，随着题⽬的不断的增加以及乱序功能的补充，都会让这段代码变得越来越混乱。



## 使用原型模式 重构代码

+ 克隆对象处理类

```java
@Setter
@Getter
public class QuestionCollection implements Cloneable{

    private String username;
    private String number;

    private ArrayList<ChoiceQuestion> choiceQuestions = new ArrayList<>();
    private ArrayList<FieldQuestion> fieldQuestions = new ArrayList<>();

    public QuestionCollection append(ChoiceQuestion choiceQuestion){
        choiceQuestions.add(choiceQuestion);
        return this;
    }

    public QuestionCollection append(FieldQuestion fieldQuestion){
        fieldQuestions.add(fieldQuestion);
        return this;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        QuestionCollection questionCollection = (QuestionCollection) super.clone();
        questionCollection.choiceQuestions = (ArrayList<ChoiceQuestion>) choiceQuestions.clone();
        questionCollection.fieldQuestions = (ArrayList<FieldQuestion>) fieldQuestions.clone();

        // 题目乱序
        Collections.shuffle(questionCollection.choiceQuestions);
        Collections.shuffle(questionCollection.fieldQuestions);

        return questionCollection;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();

        sb.append("学生：" + username);
        sb.append("\r\n学号：" + number);
        sb.append("\r\n选择题：：");
        choiceQuestions.forEach(data -> {
            sb.append("\r\n" + data.getName());
            for (String s : data.getOption().keySet()) {
                sb.append("\r\n" + s + ":" + data.getOption().get(s));
            }
        });

        sb.append("填空题：");
        for (FieldQuestion question : fieldQuestions) {
            sb.append("\r\n" + question.getName());
        }

        return sb.toString();
    }
}
```

+ 仿造试题添加对象

```java
public class QuestionPutUtil {

    private QuestionCollection questionCollection;

    public QuestionPutUtil() {
        questionCollection = new QuestionCollection();

        // 本来是数据库读取的内容，我们没有应用数据库，所以直接手动创建
        Map<String,String> q1 = new HashMap<>();
        q1.put("A","1");
        q1.put("B","2");
        q1.put("C","3");
        q1.put("D","4");

        Map<String,String> q2 = new HashMap<>();
        q2.put("A","STRING");
        q2.put("B","INTEGER");
        q2.put("C","CHAR");
        q2.put("D","LIST");

        Map<String,String> q3 = new HashMap<>();
        q3.put("A","Thread");
        q3.put("B","Excutors");
        q3.put("C","ThreadPool");
        q3.put("D","ThreadLocal");

        Map<String,String> q4 = new HashMap<>();
        q4.put("A","Binary");
        q4.put("B","Hex");
        q4.put("C","Hash");
        q4.put("D","VERBOSE");

        questionCollection.append(new ChoiceQuestion("1+1=?",q1));
        questionCollection.append(new ChoiceQuestion("字符串的数据类型是？",q2));
        questionCollection.append(new ChoiceQuestion("线程公用的数据存放对象是？",q3));
        questionCollection.append(new ChoiceQuestion("Java 反编译class文件，我们要使用哪个参数？",q4));

        questionCollection.append(new FieldQuestion("埋在地下一千年的酒，出来以后叫什么?","酒精"));
        questionCollection.append(new FieldQuestion("有一只猪，它走啊走啊，走到了英国,结果他变成了什么?","pig"));
        questionCollection.append(new FieldQuestion("上课老师抽查背课文,小猪,小狗,小猫都举手了,老师会叫谁?","小狗，因为旺旺仙贝"));
        questionCollection.append(new FieldQuestion("蝴蝶, 蚂蚁, 蜘蛛, 蜈蚣,他们一起工作,最后哪一个没有领到酬劳?","蜈蚣，因为无功不受禄"));

    }

    public String createPaper(String username, String number) throws
            CloneNotSupportedException {
        QuestionCollection questionBankClone = (QuestionCollection)
                questionCollection.clone();
        questionBankClone.setUsername(username);
        questionBankClone.setNumber(number);
        return questionBankClone.toString();
    }
}
```



现在都准备好了，那么我们来测试一下

+ Test

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test() throws CloneNotSupportedException {
        QuestionPutUtil questionPutUtil = new QuestionPutUtil();
        log.info(questionPutUtil.createPaper("隔壁老王","123333"));
    }
}
```

运行的结果

```console
16:46:59.511 [main] INFO  c.wills.java.prototype.upgrade.Test - 学生：隔壁老王
学号：123333
选择题：：
Java 反编译class文件，我们要使用哪个参数？
A:Binary
B:Hex
C:Hash
D:VERBOSE
字符串的数据类型是？
A:STRING
B:INTEGER
C:CHAR
D:LIST
线程公用的数据存放对象是？
A:Thread
B:Excutors
C:ThreadPool
D:ThreadLocal
1+1=?
A:1
B:2
C:3
D:4填空题：
埋在地下一千年的酒，出来以后叫什么?
有一只猪，它走啊走啊，走到了英国,结果他变成了什么?
上课老师抽查背课文,小猪,小狗,小猫都举手了,老师会叫谁?
蝴蝶, 蚂蚁, 蜘蛛, 蜈蚣,他们一起工作,最后哪一个没有领到酬劳?
```



总结：

1.以上的实际场景模拟了原型模式在开发中᯿构的作⽤，但是原型模式的使⽤频率确实不是很⾼。如果有⼀些特殊场景需要使⽤到，也可以按照此设计模式进⾏优化。

2.另外原型设计模式的优点包括；便于通过克隆⽅式创建复杂对象、也可以避免重复做初始化操作、不需要与类中所属的其他类耦合等。但也有⼀些缺点如果对象中包括了循环引⽤的克隆，以及类中深度使⽤对象的克隆，都会使此模式变得异常麻烦。