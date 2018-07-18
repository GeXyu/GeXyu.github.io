---
title: 自己手写数据库连接池和SpringData
comments: true
tags:
  - 自己手写数据库连接池和SpringData
categories:
  - 自己手写数据库连接池和SpringData
url: 387.html
id: 387
date: 2017-06-23 18:27:45
---

### **GitHub地址：[传送门](https://github.com/GeXyu/data2)**

实际开发中最常用到连接池，连接池是创建和管理一个连接的缓冲池的技术，这些连接准备好被任何需要它们的线程使用。`连接池的出现可以减少资源的频繁请求和释放。`

连接池实现的核心就是：`当用户关闭连接时，我们把连接返回到池中，好使其他线程使用，而并非真正的关闭`。 从这个角度分析，我们只需要重写连接的close方法，就可以实现一个简单的连接池。可以使用包装设计模式或者是代理模式，重写close方法即可。 但是包装设计模式需要重写所有的类，比较麻烦，所以我使用的是代理模式。（具体代码可以参考`cn.xiuyu.pool`）  
`使用连接池只需要在classpath下面加入jdbc.properties文件配置属性即可`

jdbcDriver=com.mysql.jdbc.Driver
userName=root
password=tiger
url=jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8
MAXPOOLSIZE=15
InitPOOLSIZE=10

Spring Data是一个用于简化数据库访问，并支持云服务的开源框架。其主要目标是使得对数据的访问变得方便快捷，并支持map-reduce框架和云计算数据服务。  
`当spring data与jpa子项目一起使用时只需要写一个dao接口，继承一个jparepository接口。spring data就会自动帮你生成一个代理dao类,并自动生成sql语句运行并返回结果，当有复杂操作时在方法上面添加@Query注解 就可以自动sql语句。这也是我实现的核心。`

#### **仿照spring data实现了自动生成代理类，支持简单的CURD，分页,和一些自定的复杂查询**

#### **如何使用？**

写一个UserResponse类，将UserRespons继承`Repository`,再一个实体类User用于封装数据,然后在测试类使用`ResponseFactory.getInstance()`获取ResponseFactory的工厂实例，然后再使用工厂方法`createResponse`创建dao代理对象，然后就可以进行快乐的操作了.(这也是我设计不合理的地方，很狗血，这也是我不想继续写下去的原因)

##### UserResponse类

public interface UserResponse extends Repository<User>{
	@Delete("delete from user where id=?")
	public void deletebyid(int id);
	@Query("from User where id=?")
	public User QuerybyID(int id);
	
	@Updata("update user set name=?,password=? where id=?")
	public void updatebyID(String name,String password,int id);
}

##### 封装的方法使用

public class TestOrm {
	//获取代理的类和封装的实体类
	ResponseFactory factory = ResponseFactory.getInstance();
	UserResponse createResponse = factory.createResponse(UserResponse.class, User.class);
	
	@Test
	//添加用户
	public void add() throws Exception{
		for(int i=0;i<100;i++){
			User u = new User(i+"",i+"张三","123123");
			createResponse.insert(u);
		}
	}
	@Test
	//更新用户
	public void update() throws Exception{
		User u = new User("1","1","1");
		createResponse.update(u);
	}
	@Test
	//删除用户
	public void delete() throws Exception{
		createResponse.delete(1);
	}
	@Test
	//查询用户
	public void query() throws Exception{
		User u = createResponse.query(1);
		System.out.println(u);
	}
	
	@Test
	//获取总数
	public void count() throws SQLException{
		System.out.println(createResponse.Querycount());
	}
	
	@Test
	//分页
	public void page() throws InstantiationException, IllegalAccessException, SQLException{
		//从第0叶开始查询 查询3页
		PageEntity<User> queryBypage = createResponse.QueryBypage(new PageEntity<User>(0, 3));
		List<User> entity = queryBypage.getEntity();
		System.out.println(entity.size());
		for(User u:entity){
			System.out.println(u);
		}
	}
	
	
	@Test
	//通过@Query注解查询
	public void querybysql() {
		User querybyID = createResponse.QuerybyID(2);
		System.out.println(querybyID);
	}
	@Test
	//通过@Delete注解删除
	public void deletebysql(){
		createResponse.deletebyid(1);
	}
	
	@Test
	//通过@Update注解更新
	public void updatebysql(){
		createResponse.updatebyID("123", "123", 2);
	}
	
}