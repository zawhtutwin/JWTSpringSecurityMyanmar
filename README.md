စစချင်း pom.xml ထဲမှာ ဒီ dependency ကို ထည့်ပါ။

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

ပြီးရင်တော့ JwtAuthenticationFilter ကို ဟောသလို ရေးပြီး filter package ထဲထည့်ထားပါ။ JWT အကြောင်းကို နောက် chapter မှာ ရှင်းပြပါ့မယ်။

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final String SECRET = "my-secret-key";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            return;
        }

        try {
            String token = authHeader.substring(7);
            Claims claims = Jwts.parser()
                                .setSigningKey(SECRET)
                                .parseClaimsJws(token)
                                .getBody();

            // Set authentication if needed
            String username = claims.getSubject();
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(username, null, List.of());

            SecurityContextHolder.getContext().setAuthentication(authentication);
        } catch (Exception e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            return;
        }

        filterChain.doFilter(request, response);
    }
}

```

ဒါကတော့ Security Config ဖြစ်ပါတယ်။ ဒါကို base package မှာပဲထည့်ပါ။

```java
package com.clinic.app;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

import com.clinic.app.filters.JwtAuthenticationFilter ;

@Configuration
public class SecurityConfig {

     @Autowired 
     JwtAuthenticationFilter apiKeyAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(apiKeyAuthFilter, org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

}
```

ဒါကတော့ Bearer token ထုတ်ပေးမယ့် api/auth endpoint ပါ။ localhost:8080/api/auth ကို user name နဲ့ password နဲ့ ပေးလိုက်ရင် ထွက်လာမှာပါ။ ဒီနေရာမှာ username နဲ့ password ကို လွယ်ကူအောင် အသေရေးထားပါတယ်။ SECRET လည်း ထို့အတူပါပဲ။

```java
@RestController
@RequestMapping("/api")
public class AuthController {

    private final String SECRET = "my-secret-key";

    @PostMapping("/auth")
    public ResponseEntity<?> authenticate(@RequestBody AuthRequest request) {
        // In real apps, you would validate username/password
        if ("user".equals(request.getUsername()) && "pass".equals(request.getPassword())) {
            String token = Jwts.builder()
                    .setSubject(request.getUsername())
                    .signWith(SignatureAlgorithm.HS256, SECRET)
                    .compact();

            return ResponseEntity.ok(Collections.singletonMap("token", token));
        } else {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid credentials");
        }
    }
}

```

ပြီးရင် postman က နေ http://localhost:8080/api/auth ကို username,password ပေးပြီး POST request ခေါ်ကြည့်ပါ။ Bearer Token ရလာပါလိမ့်မယ်။

```
                .requestMatchers("/api/auth").permitAll()
                .anyRequest().authenticated()
```
ဒီ code ထဲမှာ ပါတဲ့ အတိုင်း /api/auth က လွဲရင် ကျန်တဲ့ route တွေအကုန်လုံးကို Spring Security က စစ်ပါတယ်။ Authenticate လုပ်ပါတယ်။ Bearer Token ပါမှ ပေးဝင်တဲ့ သဘောပါ။

JWT အကြောင်း နဲနဲပြောပြပါ့မယ်။ JWT မှာ Header, Payload နဲ့ Signature ဆိုပြီး သုံးမျိုးပါပါတယ်။

ဒါက Header ပါ
```json
{
  "alg": "HS256",
  "typ": "JWT"
}

```
ဒါက payload

```
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "exp": 1715467200
}
```

ဒါက Payload ကို Java မှာ create လုပ်တဲ့နည်း က ခုနက example ထဲ မှာ ပါပပြီးသားပါ။ username နဲ့ password မှန်ရင် jwt ကို generate လုပ်ပြီး token ထုတ်ပေးပါတယ်။

```java
            String token = Jwts.builder()
                    .setSubject(request.getUsername())
                    .signWith(SignatureAlgorithm.HS256, SECRET)
                    .compact();
```
ဒီတော့ JWT ရဲ့ Format က 

```text
<base64url-encoded-header>.<base64url-encoded-payload>.<base64url-encoded-signature>
```

နောက်ဆုံးက signature ကို token ကို ဘယ်သူမှ ပြင်ပြီးမသုံးဖူးဆိုတာ အာမခံတဲ့ အနေနဲ့ သုံးပါတယ်။ သူ့ရဲ့အလုပ်လုပ်ပုံက header နဲ့ payload ကို encode လုပ်ပြီး secret key နဲ့ sign လုပ်ထားတာပါ။ example ဒါမျိုးပါ။

```text

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

