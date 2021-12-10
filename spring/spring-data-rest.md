# Spring Data REST
#### Doc-s are here - https://docs.spring.io/spring-data/rest/docs/current/reference/html
#### Next examples represented via **_Kotlin_** 

Topics:

- [Change base path for resources](#_change-base-path-for-resources_)
- [Basic example](#_basic-example_)
- [Search](#_search_)
- [Resource with inner resource](#_resource-with-inner-resource_)
- [Jackson custom serialization](#_jackson-custom-serialization-on-post_)
- [Expose ID-s](#_expose-id-s_)
- [Restrictions for HTTP methods](#_restrictions-for-http-methods_)

### _Change base path for resources_
You can apply the base path for all **DataRepositories** via the spring boot's application file

E.g. application.properties
```properties
spring.data.rest.basePath=/resources
# strategies to expose data repositories (all|default|visibility|annotated)
spring.data.rest.detection-strategy=annotated
```
Also, you can create a manual configuration with `RepositoryRestConfigurer` class

```kotlin
@Component
class CustomizedRestMvcConfiguration : RepositoryRestConfigurer {
    override fun configureRepositoryRestConfiguration(config: RepositoryRestConfiguration, cors: CorsRegistry) {
        config.setBasePath("/resources")
		// other additional configuration like change body on create/update action...
    }
}
```

### _Basic Example_

Create entity and repository. If you are using `JpaRepository` Spring adds pagination/sort things (`http://resource-name/?size=5&page=2`), `CrudRepository` provides only basic entities.

```kotlin
@RepositoryRestResource
interface PersonRepository : JpaRepository<Person, Long>

@Table(name = "person")
@Entity
open class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id", nullable = false)
    open var id: Long? = null

    @Column(name = "first_name", length = 50)
    open var firstName: String? = null

    @Column(name = "last_name", length = 50)
    open var lastName: String? = null
}
```

### _Search_

```kotlin
	@RestResource(path = "nameStartsWith", rel = "myResourceName")
	fun findByFirstNameStartsWith(@Param("name") name: String, p: Pageable): Page<Person>
```
The result will be (`http://localhost:8080/resources/persons/search`):

```json
{
	"_links": {
		"myResourceName": {
			"href": "http://localhost:8080/resources/persons/search/nameStartsWith{?name,page,size,sort}",
			"templated": true
		},
		"self": {
			"href": "http://localhost:8080/resources/persons/search"
		}
	}
}
```
### _Resource with inner resource_
Let's add to our person an address entity with the `@OneToOne` binding, using interface:
```kotlin
    // class Person
    @OneToOne(targetEntity = BaseAddress::class, orphanRemoval = true)
    open var address: Address? = null
```
The address entity and the interface:
```kotlin
interface Address {
    var street: String?
    var country: String?
}

@Table(name = "base_address")
@Entity
open class BaseAddress : Address {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id", nullable = false)
    open var id: Long? = null

    @Column(name = "street", length = 50)
    override var street: String? = null

    @Column(name = "country", length = 100)
    override var country: String? = null
}
```
If you have the Address repository, and it's exposed like `PersonRepository` you'll get a link to the resource where you can get address information

If `Address` is not exposed all fields will be attached to the Person entity

### _Jackson custom serialization on POST_
The preceding example works great when we are trying to `GET` data, but on the `POST` method we will receive an exception because Jackson doesn't know which implementation it should use

To avoid this scenario we'll add a custom configuration

```kotlin
@Component
class CustomizedRestMvcConfiguration : RepositoryRestConfigurer {
    override fun configureJacksonObjectMapper(objectMapper: ObjectMapper) {
        objectMapper.registerModule(object : SimpleModule("AddressModule") {
            override fun setupModule(context: SetupContext) {
                context.addAbstractTypeResolver(
                    SimpleAbstractTypeResolver()
                        .addMapping(Address::class.java, BaseAddress::class.java)
                )
            }
        })
    }
}
```

### _Projections_
In case when you want to provide specific fields of your entity, you can add `Projections`

Let's try to add only the last name of our Person's entity:

```kotlin
@Projection(name = "last-name", types = [Person::class])
interface OnlyLastName {
    val lastName: String
}
```

Now when we'll try to get our resource we will see that we have a new additional link `http://localhost:8080/resources/persons/1{?projection}`

Thus, we can provide our projection with the query `/perons/1?projection=last-name` and we will get our entity only with last name

If you want to apply your projection for `GET` all entities, just add:

```kotlin
@RepositoryRestResource(excerptProjection = OnlyLastName::class)
interface PersonRepository : JpaRepository<Person, Long>
```

The result will be `[{"lastName" : "Ivanov"}, ...]`

### _Expose ID-s_

To provide ID-s we can additionally configure our `RepositoryRestConfigurer`

```kotlin
@Component
class CustomizedRestMvcConfiguration(
    private val entityManager: EntityManager
) : RepositoryRestConfigurer {

    override fun configureRepositoryRestConfiguration(config: RepositoryRestConfiguration, cors: CorsRegistry) {
        config.exposeIdsFor(*entityManager.metamodel.entities.map { it.javaType }.toTypedArray())
    }
}
```

Of course, you can add one by one entity, but this trick is useful

:warning: If you do expose for all entities make sure your configured security right

### _Restrictions for HTTP methods_

There are two ways how you can set up HTTP methods for your entity
1) Change visibility in a repository
2) Use security

The first option is really simple. All you need to do is put the `@RestResource(exported = false)` annotation, e.g.:

Let's try to remove the `GET` access for all our persons

```kotlin
@RepositoryRestResource
interface PersonRepository : JpaRepository<Person, Long> {
    @RestResource(exported = false)
    override fun findAll(pageable: Pageable): Page<Person>
}
```

That's it! And if we try to use this url `http://localhost:8080/resources/persons` we will get in our logs - `Resolved [org.springframework.web.HttpRequestMethodNotSupportedException: Request method 'GET' not supported]`

Thus, if you need to restrict access you can remove its expose (applicable for all http methods)

**Using security**

It'll work in the same way as for exposing. Just add `@PreAuthorize("hasRole('ROLE_USER')")` to the class or override method members

Or you can apply restrictions on a path, like for all other resources in `Spring Security`