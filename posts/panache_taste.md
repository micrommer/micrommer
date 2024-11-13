# Panache taste

### I was reading Quarkus docs and something caught my eye and it was Panache. As they defined it in the document, It's actually Hibernate ORM with panache that focuses on making your entities trivial and fun to write in Quarkus and this really is!

If the description was a little confusing, As far as I understand, Panache is not an ORM, it is a kind of wrapper that, makes it easier to work with some techs like Hibernate and even REST, Panache simplified them. In the background, Panache deals with EntityManager (The Hibernate implementation).

### Let's embark on the short journey

To have a good playground, it's better to have an appropriate environment, I will be using the Quarkus CLI but other option like the website or Intellij Quarkus plugin is available too to build it up.

To create a Quarkus app, simply run:

```
> quarkus create app io.micrommer:panache-ground:0.1  --gradle --java=1
7
```

And add desired dependencies as follow:

```
> quarkus ext add quarkus-hibernate-orm-panache
> quarkus ext add quarkus-jdbc-h2
```

Now we have them all. It's time to unleash the Panache entity by creating a Client entity. To have a Panache entity you should extend PanacheEntity or PanacheBaseEntity, but wait, what are the differences?

PanacheEntity extends PanacheBaseEntity and offers an auto-generated ID field and overrides PanacheBaseEntity .toString() method to append its own ID. So if you are happy with an auto-generated Long field as your entity's ID go with PanacheEntity, otherwise go with PanacheBaseEntity, an easy trade-off. It was the first simplification that Panache promised.

Client Code:

```
@Entity
public class Client extends PanacheEntity {
    public enum Status {
        ACTIVE, DEACTIVATE, BLOCKED
    }

    public String name;
    @Enumerated(EnumType.STRING)
    public Status status;
    public LocalDateTime createdDate;
}
```

At first look, everything is all right, but if you look at the entity's fields accessibility scope, something is wrong! They are public! Where is encapsulation? Don't worry here is the second simplification, Panache will generate all accessor methods for you, so you could be free from Lombok (at least in the Panache territory). If you want to use a private accessor for the fields, it is possible to define them as private but you should create the accessor method by yourself or ask it from Lombok.

House Code:

```
@Entity
public class House extends PanacheEntityBase {
    public enum Status {
        AVAILABLE, UNAVAILABLE, SOLD
    }
    @Id
    public BigInteger id;
    @ManyToOne
    public Client owner;
    public String address;
    public Integer area;
    public Integer price;
    @Enumerated(EnumType.STRING)
    public Status status;
}
```

In the House entity, we need a BigInteger as ID and it is not an auto-generated field.

Recommended by LinkedIn

Client Controller

```
@Path("/clients")
public class ClientController {
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public List<String> getClient() {
        final List<Client> clients = Client.listAll();
        return clients.stream().map(PanacheEntity::toString).collect(Collectors.toList());
    }

    @POST
    @Transactional
    public Response createClient(@QueryParam("name") String name) {
        final var client = new Client(Objects.isNull(name) ? RandomString.make(4) : name, Client.Status.ACTIVE, LocalDateTime.now());
        client.persist();
        if (client.isPersistent())
            return ResponseBuilderImpl.ok().build();
        else return Response.status(Response.Status.INTERNAL_SERVER_ERROR).build();
    }
}
```

And simply we could use the entity without any repository and Injection! And the shiny aspect of Panache just appeared. It offers an Active Record pattern as default. It could be a lovely alternative to the repository pattern that is usually used in spring data but if you don't like the Active Record pattern, panache also offers PanacheRepository and PanacheRepositoryBase with the same differences, PanacheRepository assumes your ID is Long but PanacheRepositoryBase ask you for the ID type. So It could be a third simplification, I mean the Active record, reduces the code length if you like.

Now you have access to a bunch of pre-defined static methods that are so comprehensive and cover lots of use cases.

For example, an endpoint to fetch all blocked clients could be like this:

```
@GET
@Path("/blocked")
public List<Client> getBlockedClient() {
    List<Client> blockedClient= Client.list("status", Client.Status.BLOCKED);
    return blockedClient;
}
```

Or simply do pagination:

```
@GET
@Path("/blocked/first-page")
public List<Client> getBlockedPaginatedClient() {
    List<Client> blockedClient = Client.find("status", Client.Status.BLOCKED).firstPage().list();
    return blockedClient;
}
```

Probably it is not a best practice to hardcoding a query in the controller, if you looking for a place to write down your query to use across the application the best place is in the entity itself.

In Client entity:

```
public static List<Client> getAllBlockedUserInPeriod(LocalDateTime from, LocalDateTime to) {
    return list("status = :status and createdDate between :from and :to",
            Parameters.with("status", Status.BLOCKED).and("from", from).and("to", to));
}
```

It was a brief introduction, and there is much more stuff to deal with if you like. something that I should mention before the end, You can not use Mockito with Panache active record pattern and You should use quarkus-panache-mock instead for mocking purposes.

You can find the code [here](https://www.linkedin.com/redir/redirect?url=https%3A%2F%2Fgithub%2Ecom%2Fmicrommer%2Fpanache-ground&urlhash=Ap6T&trk=article-ssr-frontend-pulse_little-text-block) and the documentation [here](https://www.linkedin.com/redir/redirect?url=https%3A%2F%2Fquarkus%2Eio%2Fguides%2Fhibernate-orm-panache&urlhash=dMu5&trk=article-ssr-frontend-pulse_little-text-block) .

---

Panaché in word

> A beer cocktail made of pale beer and lemonade, panaché is a refreshing treat to enjoy on a hot summer day. Very popular in France and Italy, it is also known as shandy or radler(especially in German-speaking countries). Make your beer lighter and with a pleasant lemony taste with this panache recipe. \[[source](https://www.linkedin.com/redir/redirect?url=http%3A%2F%2Felectricbluefood%2Ecom%2Fpanache-beer-shandy%2F%23%3A%7E%3Atext%3DA%2520beer%2520cocktail%2520made%2520of%2Cin%2520German%252Dspeaking%2520countries%29%2E&urlhash=9BmU&trk=article-ssr-frontend-pulse_little-text-block) \]
