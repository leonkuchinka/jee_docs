## JPA - Beziehungen



![pic](class%20diagram.png)

##### 1:n unidirektional ohne assoz. Tabelle 

```java
@OneToMany
@JoinColumn(name="s_id")
List<Laptop> laptops;
```

###### oder

```java
							    @ManyToOne
							    Sudent student;
```

##### 1:n unidirektional mit assoz. Tabelle 

```java
@OneToMany
List<Laptop> laptops;
```

##### 1:n bidirektional ohne assoz. Tabelle

```java
							    @ManyToOne
							    Student student;
@OneToMany(mappedBy = "student")
List<Laptop> laptops;
```

##### 1:n bidirektional mit assoz. Tabelle

```java
							    @ManyToOne
							    Student student;
@OneToMany
List<Laptop> laptops;
```

##### n:m bidirektional

```java
							    @ManyToMany
							    List<Student> students;
@ManyToMany(mappedBy = "students")
List<Laptop> laptops;
```

##### n:m unidirektional

```java
@ManyToMany
@JoinTable(name="Student_Laptop", 
           joinColumns = @JoinColumn("id"),
           inverseJoinColumns = @JoinColumn("id"))
List<Laptop> laptops;

```



### Zu beachten

###### Bei bidirektionalen Beziehungen müssen die Listen immer synchron gehalten werden:

- bei OneToMany: 

  ```java
					      public void setStudent(Student student) {
						  if(this.student != null && this.student.getLaptops().contains(this))
						      this.student.removeLaptop(this);
						  if(act != null && 
						     !student.getLaptops().contains(this))
						      student.addLaptop(this);
						  this.student = student;
					      }
  
  public void addLaptop(Laptop laptop) {
      if(!this.laptops.contains(laptop))
          this.laptops.add(laptop);
      if(laptop.getStudent() != this)
          laptop.setStudent(this);
  }
  public void removeLaptop(Laptop laptop) {
      if(this.laptops.contains(laptop))
          this.laptops.remove(laptop);
      if(laptop.getStudent() == this)
          laptop.setStudent(null);
  }
  ```

  

- bei ManyToMany (auf beiden Seiten:)

  ```java
  public void addLaptop(Laptop laptop) {
      if(!this.laptops.contains(laptop))
          this.laptops.add(laptop);
      if(!laptop.getStudents().contains(this))
          laptop.addStudents(this);
  }
  public void removeLaptop(Laptop laptop) {
      if(this.laptops.contains(laptop))
          this.laptops.remove(laptop);
      if(laptop.getStudents().contains(this))
          laptop.removeStudents(this);
  }
  
  ```

  ###### Converter für die Datenbank

  ```java
  @Converter(autoApply = true)
  public class LocalDateConverter implements AttributeConverter<LocalDate, Date> {
      @Override
      public Date convertToDatabaseColumn(LocalDate localDate) {
          return Date.valueOf(localDate);
      }
  
      @Override
      public LocalDate convertToEntityAttribute(Date date) {
          return date.toLocalDate();
      }
  }
  
  
  ```

  ###### persistence.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">
      <persistence-unit name="dbPU" transaction-type="JTA">
          <jta-data-source>java:jboss/datasources/dbDS</jta-data-source>
          <properties>
		<!--<property name="hibernate.hbm2ddl.auto" value="create-drop"></property>
		<property name="hibernate.show_sql" value="true"></property>
		<property name="hibernate.format_sql" value="true"></property>
		<property name="hibernate.trancsaction.flush_before_completion" value="true"></property>
		<property name="hibernate.dialect" value="org.hibernate.dialect.DerbyDialect"></property>-->
		<property name="javax.persistence.schema-generation.database.action" value="drop-and-create"/>
		<property name="javax.persistence.schema-generation.scripts.action" value="drop-and-create"/>
		<property name="javax.persistence.schema-generation.scripts.create-target" value="sampleCreate.ddl"/>
		<property name="javax.persistence.schema-generation.scripts.drop-target" value="sampleDrop.ddl"/>
          </properties>
      </persistence-unit>
  </persistence>
  
  ```

  

## REST

###### XML-Adapter für XML-Binding

```java
public class LocalDateAdapter extends XmlAdapter<String, LocalDate> {

    @Override
    public LocalDate unmarshal(String v) throws Exception {
        return LocalDate.parse(v, DateTimeFormatter.ISO_DATE);
    }

    @Override
    public String marshal(LocalDate v) throws Exception {
        return v.format(DateTimeFormatter.ISO_DATE);
    }
}

```

-> Verwendung:

```java
@XmlJavaTypeAdapter(XmlAdapter.class)
LocalDate date;

```

###### Binding Cycles verhindern

```java
@JsonbTransient
@XmlTransient

```

###### InitBean

```java
@ApplicationScoped
public class InitBean {

    @PersistenceContext
    EntityManager em;

    @Transactional
    private void init(@Observes @Initialized(ApplicationScoped.class) Object init) {
        System.out.println("******************** init ********************");
        em.persist(....);
	}
}

```

###### RestConfig

```java
@ApplicationPath("api")
public class RestConfig extends Application {
}

```

###### Response Codes:

```javascript
GET
	- 200 OK wenn erfolgreich
	- 404 Not Found wenn Resource nicht gefunden

POST
	- 201 Created wenn erfolgreich
   	- 422 Unprocessable Entity wenn nicht erfolgreich

PUT
	- 200 OK oder 204 No Content wenn erfolgreich
   	- 404 Not Found wenn Resource nicht gefunden
    
DELETE
	- 200 OK oder 204 No Content wenn erfolgreich
    	- 404 Not Found wenn Resource nicht gefunden

```



## Jersey

##### unmarshall

```java
Student student = new Student();
student.setName(json.getString("name"));

JsonArray laptops = json.getJsonArray("laptops");
for (JsonValue val: laptops) {
    JsonObject obj = val.asJsonObject();
    Laptop laptop = new Laptop(obj.getString("name"));
	student.addLaptop(laptop);
}
return student;

```

##### marshall

```java
JsonObjectBuilder studentObj = Json.createObjectBuilder();
studentObj.add("id", student.getId());

JsonArrayBuilder laptops = Json.createArrayBuilder();
em.createNamedQuery("Laptop.findByStudent", Laptop.class)
    .setParameter(1, student.getId())
    .getResultList()
    .stream()
    .foreach(laptop -> {
        JsonObjectBuilder laptopObj = Json.createObjectBuilder();
        laptopObj.add("id", laptop.getId());
        laptopObj.add("name", laptop.getName());
        laptops.add(laptopObj.build());
    });

studentObj.add("laptops", laptops);

return studentObj.build();


```

## JUnit

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.containsString;
import static org.hamcrest.Matchers.greaterThan;
import static org.hamcrest.Matchers.is;

@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class MyAppST {
    private static Client client;
    private static WebTarget target;

    @BeforeClass
    public static void init(){
        client = ClientBuilder.newClient();
        target = client.target("http://localhost:8080/myapp/api");
    }
    
    @Test
    public void test01_Something(){
        JsonObject response = target.path("/something/1")
            .request(MediaType.APPLICATION_JSON)
            .get(JsonObject.class);
        assertThat(response.getStatus(), is(200));
        assertThat(response.getInt("id"), is(1));
        assertThat(response.getString("somethiing"), is("something"));
	}
    
    @Test
    public void test03_CreateSomething(){
        JsonObject obj = ....
            Response response = this.target.path("/something")
            .request()
            .post(Entity.json(obj));
        assertThat(response.getStatus(), is(201));
        JsonObject createdObj = this.client.target(response.getLocation())
            .request(MediaType.APPLICATION_JSON)
            .get()
            .readEntity(JsonObject.class);
        assertThat(response.getStatus(), is(200));
        assertThat(createdObj.getInt("something"), is("something"));
	}
    @Test
    public void test06_DeleteSomethingTwice(){
        Response response = this.target
            .path("/something/" + id)
            .request()
            .delete();
        assertThat(response.getStatus(), is(200));
        response = this.target
            .path("/something/" + id)
            .request()
            .delete();
        assertThat(response.getStatus(), is(200));
    }
}


```

