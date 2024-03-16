# Demo of Spring Security internal flow

## <u>Authorization Filter</u>

restricts access to url that user is trying to access. If it is public url, the response will be displayed to user straigh away without asking credentials. But if it is secured url, Authorization filter will stop acess to that url and will re-direct request to the next filter available inside Security Filter Chain. We have doFIlter() method that will take help from Authorization Manager to check wheater a url is public or secured url and based on that it will allow or deny access to the url.

```java
@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
			throws ServletException, IOException {

		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response = (HttpServletResponse) servletResponse;

		if (this.observeOncePerRequest && isApplied(request)) {
			chain.doFilter(request, response);
			return;
		}

		if (skipDispatch(request)) {
			chain.doFilter(request, response);
			return;
		}

		String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
		request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
		try {
			AuthorizationDecision decision = this.authorizationManager.check(this::getAuthentication, request);
			this.eventPublisher.publishAuthorizationEvent(this::getAuthentication, request, decision);
			if (decision != null && !decision.isGranted()) {
				throw new AccessDeniedException("Access Denied");
			}
			chain.doFilter(request, response);
		}
		finally {
			request.removeAttribute(alreadyFilteredAttributeName);
		}
	}
```

If we try to access secured url then next filter is DefaultLoginPageGeneratingFilter which will show us login page. That page is build using this filter using html. Once user entered their credentials like username and pasword next filter is UsernamePasswordAuthenticationFilter. This filter has attemptAuthentication method. This filter extracts username and password from HttpServletRequest. And with that username and password its going to form object of UsernamePasswordAuthenticationToken. This class extends AbstractAuthenticationToken that implements Authentication interface. Then this UsernamePasswordAuthenticationToken object is going to be hand over to Authentication Manager by calling authenticate method:

```java
@Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException {
		if (this.postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
		}
		String username = obtainUsername(request);
		username = (username != null) ? username.trim() : "";
		String password = obtainPassword(request);
		password = (password != null) ? password : "";
		UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,
				password);
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```

We know that Authentication Manger is an interface so inside Spring Security framework we have a class called Provider Manager which is implementation of Authentication Manager and inside that class we have method `authenticate()`. THe role of Authentication Manager is similar to managers which lead team od developers,testers... by coordinating will all his team members and successfully implement client requirements. So Authentication Manager or Provider Manager in this case will try to interact will all Authentication Providers available inside framework or developer defined Authentication Providers. Below is the method:

```java
@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		int currentPosition = 0;
		int size = this.providers.size();
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}
			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",
						provider.getClass().getSimpleName(), ++currentPosition, size));
			}
			try {
				result = provider.authenticate(authentication);
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException | InternalAuthenticationServiceException ex) {
				prepareException(ex, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw ex;
			}
			catch (AuthenticationException ex) {
				lastException = ex;
			}
		}
		if (result == null && this.parent != null) {
			// Allow the parent to try.
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}
			catch (ProviderNotFoundException ex) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException ex) {
				parentException = ex;
				lastException = ex;
			}
		}
		if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}
			// If the parent AuthenticationManager was attempted and successful then it
			// will publish an AuthenticationSuccessEvent
			// This check prevents a duplicate AuthenticationSuccessEvent if the parent
			// AuthenticationManager already published it
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).
		if (lastException == null) {
			lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
					new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
		}
		// If the parent AuthenticationManager was attempted and failed then it will
		// publish an AbstractAuthenticationFailureEvent
		// This check prevents a duplicate AbstractAuthenticationFailureEvent if the
		// parent AuthenticationManager already published it
		if (parentException == null) {
			prepareException(lastException, authentication);
		}
		throw lastException;
	}
```

at line 10 inside for loop its going through loop over all applicable auth providers until it sees the successful authentication or failed authentication. For example if we have 2 auth providers for login and sees success in first one, its not going to try for the second auth provider.  But if first one fails, it is going to try for second one. 

One provider that Provider Manager can invoke is Dao Authentication Provider. Inside the class that it extends AbstractUserDetailsAuthenticationProvider is authenticate method and provider manager sends the request to that method and as a result we get all actual authentication logic. Here is the method:

```java
@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				() -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));
		String username = determineUsername(authentication);
		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);
		if (user == null) {
			cacheWasUsed = false;
			try {
				user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			}
			catch (UsernameNotFoundException ex) {
				this.logger.debug("Failed to find user '" + username + "'");
				if (!this.hideUserNotFoundExceptions) {
					throw ex;
				}
				throw new BadCredentialsException(this.messages
					.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
			}
			Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
		}
		try {
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException ex) {
			if (!cacheWasUsed) {
				throw ex;
			}
			// There was a problem, so try again after checking
			// we're using latest data (i.e. not from the cache)
			cacheWasUsed = false;
			user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		this.postAuthenticationChecks.check(user);
		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}
		Object principalToReturn = user;
		if (this.forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}
		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
```

First it fatches the username, then calls method retrieveUser line 12 (which is inside DaoAuthenticationProvider):

```java
@Override
	protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			if (loadedUser == null) {
				throw new InternalAuthenticationServiceException(
						"UserDetailsService returned null, which is an interface contract violation");
			}
			return loadedUser;
		}
		catch (UsernameNotFoundException ex) {
			mitigateAgainstTimingAttack(authentication);
			throw ex;
		}
		catch (InternalAuthenticationServiceException ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
		}
	}
```

and inside that method we can see that auth provider takes help from User Details Manager or User Detail Service.

So with the help of auth providers we define actual authentication logic. 

Common requrement that the developer has is to catch user details from storage system and that is the job of User details manager/service. We also have PasswordEncoder that encodes password.

Back to the retrieveUser method, like we said dao auth provider is going to take help from one implementation of Auth Manager and that is **In Memory User Details Manager**. If we defined username and password inside application proterties for example, In Memory User Details Manager is going to catch up on that and they will be stored inside in-memory of the application. Inside this class auth manager is going to invoke method loadUserByUsername(). When it gets credentials it gives them to dao auth provider in retrieveUser, then goes to its extended class AbstactUserDetailsAuthenticationProvider and those same credentials are being passed also in some other method like additionalAuthenticationChecks method. Implementation of that method is inside DAO auth provider. This method then levarages password encoder to match presented password to actual password in storage system. If it matches the response is going to be sent back to Provider Manager

## Default security configurations

By default everything is secured every api. This behaviour is described and configured in method `defaultSecurityFilterChain(HttpSecurity http)` of class `SpringBootWebSecurityConfiguration`. THis is the stepping stone of making our own security configurations.

```java
		@Bean
		@Order(SecurityProperties.BASIC_AUTH_ORDER)
		SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
			http.authorizeHttpRequests((requests) -> requests.anyRequest().authenticated());
			http.formLogin(withDefaults());
			http.httpBasic(withDefaults());
			return http.build();
		}
```

our custom security config, using Lambda DSL:

```java
@Bean
    SecurityFilterChain defaultSecurityFilterChain (HttpSecurity http) throws Exception {
        http.authorizeHttpRequests((requests) -> requests.requestMatchers("/myAccount",
                                                                          "/myBalance",
                                                                          "myLoans",
                                                                          "myCards")
                                                         .authenticated()
                                                         .requestMatchers("/contact", "/notices")
                                                         .permitAll())
            .formLogin(withDefaults())
            .httpBasic(withDefaults());
        return http.build();
    }
```

## Configure users

### 1. way WITH withDefaultPasswordEncoder method

Instead of defining users in application.yml file, we can define it inside our application with help of `InMemoryUserDetailsManager` and `UserDetails`:

```java
@Bean
    InMemoryUserDetailsManager userDetailsManager () {
        UserDetails admin = User.withDefaultPasswordEncoder()
                                .username("admin")
                                .username("12345")
                                .authorities("admin")
                                .build();
        UserDetails user = User.withDefaultPasswordEncoder()
                               .username("user")
                               .username("12345")
                               .authorities("read")
                               .build();
        return new InMemoryUserDetailsManager(admin, user);
    }
```



Method `withDefaultPasswordEncoder` is marked as depricated, ONLY beacuse it is not considered secured for production purposes.

### 2. way with  bean PasswordEncoder

```java
@Bean
    public UserDetailsService userDetailsManager () {
        
        UserDetails admin = User.withUsername("admin")
                .password("12345")
                .authorities("admin")
                .build();
        UserDetails user = User.withUsername("user")
                .password("12345")
                .authorities("read")
                .build();
        return new InMemoryUserDetailsManager(admin, user);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder (){
        return NoOpPasswordEncoder.getInstance();
    }
```

1. Da li je bitno da znaju da li je porudzbina u proizvodnji - tj da je makar jedan proizvod trenutno u proizvodnji?
2. Da li zele da im se na jednoj strani prikazuju i ISPORUCENI i GOTOVI proizvodi?
3. Da li zele da imaju karticu Gotovi proizvodi gdje ce im se nalaziti samo prozvodi sa statusom GOTOVO (zavrsen proces proizvodnje ali nije isporucen), a odvojeno karticu Svi prozvodi gdje ce imati sve prozvode pa da mogu da filtriraju npr.
4. Da li zele sami, rucno da skidaju sa stanja sirovine ili da napravimo se to desava automatski kada se sirovina iskoristi tj kad proizvod krene u prozivodnju?
5. Da li zele da se povrsina i obim sirovine racunaju automatski (bez mogucnosti prepraljvanje povrsine) na osnovu duzina i visine? 
6. Na koliko decimala se zaokruzuje obim i povrsina?
7. Postoje statusi proizvoda: PROIZVODNJA, ZAVRSENO i ISPORUCENO. Da li zele da imaju mogucnost da menjaju bilo kad u bilo koji status. Npr: ako se klikne status ISPORUCENO da li zele da zabranimo vracanje na neki od prethodnih statusa npr PROIZVODNJA. Ili da mogu da prelaze iz bilo kog u bilo koji status?

1. Da li im je bitno da znaju i imaju informaciju o tome da li je porudzbina u proizvodnji - tj da je makar jedan proizvod trenutno u proizvodnji?
2. Da li zele da im se na jednoj strani prikazuju i ISPORUCENI i GOTOVI proizvodi?
3. Da li zele da imaju karticu Gotovi proizvodi gdje ce im se nalaziti samo prozvodi sa statusom GOTOVO (zavrsen proces proizvodnje ali nije isporucen), a odvojeno karticu Svi prozvodi gdje ce imati sve prozvode pa da mogu da filtriraju npr.
4. Da li zele sami, rucno da skidaju sa stanja sirovine ili da napravimo se to desava automatski kada se sirovina iskoristi tj kad proizvod krene u prozivodnju?
5. Da li zele da se povrsina i obim sirovine racunaju automatski na osnovu duzine, visine i kolicine? (bez mogucnosti da oni prepravljaju povrsinu i obim)
6. Na koliko decimala se zaokruzuje obim i povrsina?
7. Da li moze naknadno da se doda proizvod u porudzbinu ili je to nova porudzbina?
8. Postoje statusi proizvoda: PROIZVODNJA, ZAVRSENO i ISPORUCENO. Da li zele da imaju mogucnost da menjaju bilo kad u bilo koji status. Npr: ako se klikne status ISPORUCENO da li zele da zabranimo vracanje na neki od prethodnih statusa npr PROIZVODNJA. Ili da mogu da prelaze iz bilo kog u bilo koji status?
