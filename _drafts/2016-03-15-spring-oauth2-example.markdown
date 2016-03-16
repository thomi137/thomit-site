---
layout: post
title:  "OAuth2 mit Spring Security OAuth"
date:   2016-03-15 10:14
author: Thomas Prosser
categories:
  - Technologie
  - Spring
tags:
  - Spring Security
  - Java
image: spring-security-project.png
---

Spring als Framework ist unglaublich mächtig und hat eine eher steile Lernkurve. Will man seine Webapplikation schützen, muss man sich zudem noch mit Web Security vertraut machen, was das Ganze noch zusätzlich kompliziert. Die Entwickler von Spring geben sich dabei alle Mühe damit, die Features durch Code, Beispielprojekte und Testfälle so gut wie möglich zu dokumentieren. Dennoch gibt es verschiedene Dinge, gerade im Bereich Security, die um der kurzen Dokumentation Willen meist nicht erwähnt bleiben. Ein konkretes Beispiel:

### Das Problem ###

Weil ich geschäftlich ein REST Api zu entwickeln habe, musste ich mich auch mit Security beschäftigen. Aus verschiedenen Gründen entschied ich mich, die Infrastruktur mit Spring Boot zu entwickeln. Das ging auch recht flott, bis ich meine Endpoints sichern wollte. Mein Problem waren die User Informationen, die für die Authentifizierung gebraucht werden. Spezifisch wollte ich einen Client Credentials grant flow (etwa für ein Mobile Device) implementieren. Hierzu muss Spring Security natürlich Informationen über den Client, der sich anmeldet von irgendwo beziehen. Die entsprechenden Tutorials zeigen meist die Methode, wie man die Informationen mit einer in-Memory-Authentifizierung konfiguriert:

{% highlight java linenos %}
@Configuration
@EnableAuthorisationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter{

  //...

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

		clients.inMemory().withClient("client")
			.authorizedGrantTypes("client_credentials")
			.authorities("ROLE_CLIENT")
			.scopes("read", "write")
			.secret("secret");

	}

  //...

}
{% endhighlight %}

Dass dies höchstens für einen Prototypen ein gangbarer Weg ist, wird dann auch meist erwähnt und man findet auch Konfigurationsbeispiele mit einem JDBC basierten `ClientDetailsService`, die Dokumentation wird allerdings recht schnell dünn, wenn man seinen eigenen `ClientDetailsService` implementieren möchte. Spezifisch wollte ich:

* Spring Data Repositories wiederverwenden
* MongoDB als Backend
* Passwörter verschlüsselt auf der DB hinterlegen
* Für die Tests auf eine in-Memory Datenbank zugreifen können, für die Produktion aber eine normale DB anbinden

Nachdem ich lange und intensiv eine Lösung gesucht habe, bin ich testweise auf eine Implementierung gestossen, die ich der einfachheit halber mit einer normalen SQL DB gebaut habe (die Repository Interfaces von Spring Data abstrahieren das Ganze, so dass man einfach nur Spring Data MongoDB auf den Classpath nehmen muss, um das Analog mit Mongo zu erhalten).

### Modellklasse ###

Ich wollte verschiedene OAuth2 Grant Flows unterstützen, dafür aber nur eine Tabelle verwenden, wo entweder User oder Client Informationen abgespeichert werden

{% highlight java linenos %}
@Entity
@Table(name = "accounts")
@RestResource(exported = false)
public class Account {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	Long id;

	@JsonIgnore
	String accountId;

	@JsonIgnore
	String accountSecret;

	@JsonIgnore
	String resourceIds;

	@JsonIgnore
	String scopes;

	@JsonIgnore
	String grantTypes;

	@JsonIgnore
	String redirectUris;

	@JsonIgnore
	String authorities;

	public Account(){

	}

	public Account(String clientId, String clientSecret, String resourceIds,
			String scopes, String grantTypes, String redirectUris, String authorities) {
		this.accountId = clientId;
		this.accountSecret = clientSecret;
		this.resourceIds = resourceIds;
		this.scopes = scopes;
		this.grantTypes = grantTypes;
		this.authorities = authorities;
	}

	public Account(String clientId, String clientSecret, String resourceIds,
			String scopes, String grantTypes, String authorities) {
		this(clientId, clientSecret, resourceIds, scopes, grantTypes, "", authorities);
	}

// Getter und Setter...

}

{% endhighlight %}

### Eigene Beans ###

Die Anbindung an Spring Security geschieht dann über die jeweiligen Beans (schöne neue Java8 Welt...):

{% highlight java linenos %}

@Bean
protected UserDetailsService userDetailsService() {
  return (username) -> accountRepository
      .findByAccountId(username)
      .map(a -> new User(a.getAccountId(), a.getAccountSecret(),
          true, true, true, true, AuthorityUtils
              .commaSeparatedStringToAuthorityList(a
                  .getAuthorities())))
      .orElseThrow(() -> new UsernameNotFoundException(username));
}

{% endhighlight %}

bzw.

{% highlight java linenos %}

@Bean
public ClientDetailsService getClientDetailsService() {
return (clientId) -> accountRepository.findByAccountId(clientId)
    .map(a ->{
      BaseClientDetails baseClientDetails = new BaseClientDetails(a.getAccountId(), a.getResourceIds(), a.getScopes(), a.getGrantTypes(), a.getAuthorities(), a.getRedirectUris());
      baseClientDetails.setClientSecret(a.getAccountSecret());
      return baseClientDetails;
    })
    .orElseThrow(
        () -> new ClientNotFoundException(clientId)
    );
}

{% endhighlight %}

Mit diesen beiden Beans hat man nun die eigenen Services im Context sitzen und braucht sie eigentlich nur noch in die Security Configuration einzubinden.

### Web Security ###

Um den Password Grant Flow zu unterstützen, benötigt man natürlich auch eine User Authentifizierung. Dies geschieht in der Web Security Configuration

{% highlight java linenos %}
public class MySecurityConfiguration extends
		WebSecurityConfigurerAdapter {

 //...

  @Autowired
  public void globalUserDetails(AuthenticationManagerBuilder auth)
      throws Exception {
        auth.userDetailsService(userDetailsService())
            .passwordEncoder(passwordEncoder);
          }
  }

 //...

}

{% endhighlight %}

Wobei `passwordEncoder` ein injiziertes Feld ist, dass ebenfalls über ein Bean konfiguriert werden kann. Ähnlich verhält es sich mit dem Einbinden des `ClientDetailsService`


{% highlight java linenos %}
@Configuration
@EnableAuthorisationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter{

 //...

 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.withClientDetails(getClientDetailsService());

  }

 //...    

}

{% endhighlight %}

### Test ###

Wenn die Applikation gestartet wird, kann man über `curl -X POST http://localhost:8080/oauth/token` testen, ob die Sicherung nun funktioniert. Erwartungsgemäss antwortet die Applikation mit:

{% highlight javascript %}

{
  "error":"unauthorized",
  "error_description":"Full authentication is required to access this resource"
}

{% endhighlight %}

Das OAuth2 Protokoll erwartet eine HTTP Basic Authentication für diesen Endpoint. Ausserdem erfordert der client credentials grant type im Body des POST Requests die Angabe des Grant Types (der dann mit den in der DB hinterlegten Werten verglichen wird). `curl -X POST http://wine:secret@localhost:8080/oauth/token --data "grant_type=client_credentials"` liefert dann die Antwort die wir suchen:

{% highlight javascript %}

{
  "access_token" : "00abdf90-6aab-4734-876f-de9369654c90",
  "token_type" : "bearer",
  "expires_in" : 43199,
  "scope" : "read write"
}

{% endhighlight %}

Wir haben nun ein Access Token, das für eine gewisse Zeit gültig ist und den scope für dieses Token, den man ebenfalls auf der DB hinterlegen kann.
