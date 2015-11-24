# 1.介绍

## 1.1 Spring Boot

Spring Boot基于Spring和“习惯由于配置”原则，实现快速搭建项目的准生产框架。建议现在的Java从业者快速迁移到以Spring Boot为基础开发，这将大大降低开发的难度和极大的提高开发效率。

## 1.2 Activiti

在做企业级应用开发的时候，为了避免流程和业务的耦合，我们经常会引入工作流来解决业务中所遇到的审批相关的操作。

Activiti是一个轻量级的工作流和业务流程管理平台，它的核心是一个超快的BPMN2引擎。

## 1.3 spring-boot-starters

Spring Boot基于“习惯优于配置”的原则，为大量第三方的库提供自动配置的功能。由Spring专家Josh Long主导开发的spring-boot-starters为我们在Spring Boot下使用Activiti做了自动配置。

其中主要自动配置包括：
* 	自动创建**Activiti ProcessEngine**的Bean；
* 	所有的**Activiti Service**都自动注册为Spring的Bean；
* 	创建一个**Spring Job Executor**；
* 	自动扫描并部署位于src/main/resources/processes目录下的流程处理文件。

# 2.实战

## 2.1 流程设计

Activi为我们提供了一个基于eclipse的流程设计器，安装地址为：[http://activiti.org/designer/update/](http://http://activiti.org/designer/update/)


新建Activi项目或流程

![](http://activiti.org/userguide/images/designer.create.activiti.project.png)

我们当前模拟一个简单的工作流程，某人想加入某个公司，然后有权限审批的人审批，审批同意后将此人加入组织并输出“加入组织成功”，不同意输出“加入组织失败”。

设计的流程图如下：

![](http://microinsight.cn-hangzhou.aliapp.com/wp-content/uploads/2015/11/diagram.png)



流程源码如下：

```xml
	<process id="joinProcess" name="Join process" isExecutable="true">
		<startEvent id="startevent1" name="Start">
			<extensionElements>
				<activiti:formProperty id="personId" name="person id"
					type="long" required="true"></activiti:formProperty>
				<activiti:formProperty id="compId" name="company Id"
					type="long" required="true"></activiti:formProperty>
			</extensionElements>
		</startEvent>
		<endEvent id="endevent1" name="End"></endEvent>
		<userTask id="ApprovalTask" name="Approval Task"
        activiti:candidateUsers="${joinService.findUsers(execution)}" isForCompensation="true">
			<extensionElements>
				<activiti:formProperty id="joinApproved" name="Join Approved"
					type="enum">
					<activiti:value id="true" name="Approve"></activiti:value>
					<activiti:value id="false" name="Reject"></activiti:value>
				</activiti:formProperty>
			</extensionElements>
		</userTask>
		<sequenceFlow id="flow1" sourceRef="startevent1" targetRef="ApprovalTask"></sequenceFlow>
		<serviceTask id="AutoTask" name="Auto Task"
         		activiti:expression="${joinService.joinGroup(execution)}"></serviceTask>
		<sequenceFlow id="flow2" sourceRef="ApprovalTask" targetRef="AutoTask"></sequenceFlow>
		<sequenceFlow id="flow3" sourceRef="AutoTask" targetRef="endevent1"></sequenceFlow>
	</process>
```

流程解释：

流程最左边是开始，最右边结束，左二小人图标为用户任务(User Task)需要人参与操作，我们选择有权限的操作的人来源于Spring的bean方activiti:candidateUsers="${joinService.findUsers(execution)}"，左三齿轮图标为服务任务(Service Task),是自动执行的任务，自动调用Spring的bean方法。
##  2.2 项目搭建

pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.wisely</groupId>
	<artifactId>activiti</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>activiti</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.3.0.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
		<activiti.version>5.19.0</activiti.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>org.activiti</groupId>
			<artifactId>activiti-spring-boot-starter-basic</artifactId>
			<version>${activiti.version}</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

application.properties配置

```java
spring.jpa.hibernate.ddl-auto=update
spring.jpa.database=MYSQL
spring.datasource.url=jdbc:mysql://192.168.1.68:3306/activiti-spring-boot?characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=密码
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```


## 2.3 核心代码

### 实体类

```java
@Entity
public class Person {

    @Id
    @GeneratedValue
    private Long personId;

    private String personName;

    @ManyToOne(targetEntity = Comp.class)
    private Comp comp;

	public Person() {
	}

	public Person(String personName) {
		super();
		this.personName = personName;
	}
	//省略getter、setter方法
}
```

```java
@Entity
public class Comp {
	@Id
	@GeneratedValue
	private Long compId;
	private String compName;

	@OneToMany(mappedBy = "comp",targetEntity = Person.class)
	private List<Person> people;

	public Comp(String compName) {
		this.compName = compName;
	}

	public Comp() {
	}
//省略getter、setter方法
}
```

### dao

```java
public interface PersonRepository extends JpaRepository<Person, Long> {

	public Person findByPersonName(String personName);
}
```

```java
public interface CompRepository extends JpaRepository<Comp, Long>{

}

```

### Activiti服务

```java
@Service
@Transactional
public class ActivitiService {
	//注入为我们自动配置好的服务
    @Autowired
    private RuntimeService runtimeService;

    @Autowired
    private TaskService taskService;

    //开始流程，传入申请者的id以及公司的id
    public void startProcess( Long personId, Long compId) {

        Map<String, Object> variables = new HashMap<String, Object>();
        variables.put("personId", personId);
        variables.put("compId", compId);
        runtimeService.startProcessInstanceByKey("joinProcess", variables);
    }

    //获得某个人的任务别表
    public List<Task> getTasks(String assignee) {
        return taskService.createTaskQuery().taskCandidateUser(assignee).list();
    }
    //完成任务
    public void completeTasks(Boolean joinApproved, String taskId) {
    	Map<String, Object> taskVariables = new HashMap<String, Object>();
        taskVariables.put("joinApproved", joinApproved);
        taskService.complete(taskId, taskVariables);
    }



}
```

### Service Task服务

```java
@Service
public class JoinService {
	@Autowired
	PersonRepository personRepository;

	@Autowired
	private CompRepository compRepository;

	//加入公司操作，可从DelegateExecution获取流程中的变量
	public void joinGroup(DelegateExecution execution){
     Boolean bool = (Boolean) execution.getVariable("joinApproved");
     if(bool){
    	  Long personId = (Long) execution.getVariable("personId");
    	  Long compId = (Long) execution.getVariable("compId");
    	  Comp comp = compRepository.findOne(compId);
    	  Person person = personRepository.findOne(personId);
    	  person.setComp(comp);
    	  personRepository.save(person);
    	  System.out.println("加入组织成功");
     }else{
    	 System.out.println("加入组织失败");
     }
	}
	//获取符合条件的审批人，演示这里写死，使用应用使用实际代码
	public List<String>  findUsers(DelegateExecution execution){
		return Arrays.asList("admin","wtr");
	}

}
```

### 控制器

```java
@RestController
public class MyRestController {

    @Autowired
    private ActivitiService myService;

    //开启流程实例
    @RequestMapping(value="/process/{personId}/{compId}", method= RequestMethod.GET)
    public void startProcessInstance(@PathVariable Long personId,@PathVariable Long compId) {
        myService.startProcess(personId,compId);
    }

    //获取当前人的任务
    @RequestMapping(value="/tasks", method= RequestMethod.GET)
    public List<TaskRepresentation> getTasks(@RequestParam String assignee) {
        List<Task> tasks = myService.getTasks(assignee);
        List<TaskRepresentation> dtos = new ArrayList<TaskRepresentation>();
        for (Task task : tasks) {
            dtos.add(new TaskRepresentation(task.getId(), task.getName()));
        }
        return dtos;
    }

    //完成任务
    @RequestMapping(value="/complete/{joinApproved}/{taskId}", method= RequestMethod.GET)
    public String complete(@PathVariable Boolean joinApproved,@PathVariable String taskId){
    	myService.completeTasks(joinApproved,taskId);
    	return "ok";

    }

    //Task的dto
    static class TaskRepresentation {

        private String id;
        private String name;

        public TaskRepresentation(String id, String name) {
            this.id = id;
            this.name = name;
        }

        public String getId() {
            return id;
        }

        public void setId(String id) {
            this.id = id;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }

}
```

### 入口类

```java
@SpringBootApplication
public class ActivitiApplication {

	@Autowired
	private PersonRepository personRepository;
	@Autowired
	private CompRepository compRepository;

	public static void main(String[] args) {
		SpringApplication.run(ActivitiApplication.class, args);
	}

	//初始化模拟数据
	@Bean
	public CommandLineRunner init(final ActivitiService myService) {

		return new CommandLineRunner() {
			public void run(String... strings) throws Exception {
				if (personRepository.findAll().size() == 0) {
		            personRepository.save(new Person("wtr"));
		            personRepository.save(new Person("wyf"));
		            personRepository.save(new Person("admin"));
		        }
				if(compRepository.findAll().size() == 0){
					Comp group = new Comp("great company");
					compRepository.save(group);
					Person admin = personRepository.findByPersonName("admin");
					Person wtr = personRepository.findByPersonName("wtr");
					admin.setComp(group);wtr.setComp(group);
					personRepository.save(admin);personRepository.save(wtr);

				}
			}
		};

	}
}
```

## 2.4 演示

我们使用Chrome应用PostMan作为测试