# 📖 Spring Security-nin Detallı Arxitekturası və İş Axışı

## 📌 1️⃣ Əsas Komponentlər və İdeoloji Baza

Spring Security bir **Servlet Filter əsaslı təhlükəsizlik sistemidir**. Java web tətbiqlərində bütün HTTP request-lər **Servlet Container** tərəfindən qəbul edilir və **Filter Chain**-dən keçərək prosess olunur.
**Spring Security** bu Filter Chain-in içinə öz Filterlərini yerləşdirir.

**Əsas strukturlar:**
- FilterChainProxy
- SecurityFilterChain
- Security Filters
- SecurityContext
- SecurityContextHolder
- AuthenticationManager
- AuthenticationProvider
- Authentication
- AccessDecisionManager
- AccessDecisionVoter
- ExceptionTranslationFilter
- LogoutFilter

---

## 📌 2️⃣ Əsas Axış Mərhələləri

**📍 A) FilterChainProxy**
- Servlet container-da olan bütün request-lər ilk bu proksi class-a düşür.
- İçində bir neçə SecurityFilterChain var.
- Hər SecurityFilterChain müəyyən URL pattern-lərinə görə tətbiq olunur.
  (məsələn /api/** üçün bir chain, /admin/** üçün başqa chain)

📌 Niyə? Çünki müxtəlif endpoint-lərdə fərqli security qaydaları ola bilər.

---

**📍 B) SecurityFilterChain**
- FilterChainProxy bir chain seçdikdən sonra, həmin chain-in içindəki filter-lər sırayla işə düşür.
- Burda standart olaraq belə filterlər olur:

| Filter Adı                               | Nə İş Görür                                               |
| :--------------------------------------- | :-------------------------------------------------------- |
| **SecurityContextPersistenceFilter**     | SecurityContext-i ThreadLocal-dan oxuyur və saxlayır      |
| **UsernamePasswordAuthenticationFilter** | Form-based login üçün username/password yoxlayır          |
| **JwtAuthenticationFilter**              | JWT token yoxlayır və auth verir                          |
| **ExceptionTranslationFilter**           | Exception-ları tutub, redirect və ya error response verir |
| **FilterSecurityInterceptor**            | Authorization (icazə) yoxlaması edir                      |

Bu filterlər öz aralarında doFilter(request, response, chain) metodu ilə çalışır və bir-birinin ardınca axışı davam etdirir.

---

**📍 C) SecurityContext & SecurityContextHolder**
- Hər request üçün `SecurityContext` var.
- Bu context-in içində `Authentication` obyekt saxlanılır.
- `SecurityContextHolder` bu context-i `ThreadLocal`-da saxlayır.

Bu sayədə hər request-in öz Authentication məlumatı olur və paralel request-lər bir-birinə qarışmır.

---

**📍 D) AuthenticationManager və AuthenticationProvider**
- `UsernamePasswordAuthenticationFilter` və ya `JwtAuthenticationFilter` `authentication` etmək istəyəndə `AuthenticationManager`-ə müraciət edir.
- `AuthenticationManager` öz növbəsində bir və ya bir neçə `AuthenticationProvider` çağırır.
- Hər provider öz authentication tipini yoxlayır.
  - Məs: `DaoAuthenticationProvider` → DB-də username və password yoxlayır.
  - `JwtAuthenticationProvider` → JWT token-in doğruluğunu yoxlayır.
Authentication uğurludursa:
- `Authentication` obyektinə user məlumatı və authorities əlavə edir.
- `SecurityContextHolder`-da həmin Authentication saxlanır.

---

**📍 E) AccessDecisionManager və AccessDecisionVoter**
- Controller-a çatmazdan əvvəl `FilterSecurityInterceptor` authorization yoxlayır.
- Burada `AccessDecisionManager` işə düşür.
- `AccessDecisionManager` isə bir neçə `AccessDecisionVoter` çağırır:
  - RoleVoter
  - AuthenticatedVoter
  - WebExpressionVoter
Hər voter səs verir:
- `ACCESS_GRANTED`
- `ACCESS_DENIED`
- `ABSTAIN`

Əgər kifayət qədər `ACCESS_GRANTED` varsa, request Controller-a ötürülür.

---

**📍 F) ExceptionTranslationFilter**
Əgər authentication və ya authorization zamanı istisna baş verərsə:

- **AccessDeniedException**
- **AuthenticationException**

Bu filter bu exception-u tutub:
- ya login səhifəsinə yönləndirir
- ya JSON error response qaytarır

---

**📍 G) LogoutFilter**
Logout request-lərini qarşılayır.

- SecurityContext-i təmizləyir.
- Session-ları bağlayır.
- İstəyə uyğun redirect və ya mesaj göndərir.

---

## 📌 3️⃣ Təxminən Axış Sxemi

```css
[Client Request]
      ↓
[FilterChainProxy]
      ↓
[SecurityFilterChain (SecurityContextPersistenceFilter)]
      ↓
[UsernamePasswordAuthenticationFilter]
      ↓
[AuthenticationManager → AuthenticationProvider]
      ↓
[SecurityContextHolder.setContext()]
      ↓
[ExceptionTranslationFilter]
      ↓
[FilterSecurityInterceptor (AccessDecisionManager)]
      ↓
[Controller]
      ↓
[Response]
      ↓
[SecurityContextPersistenceFilter (clearContext)]
      ↓
[Client]
```

---

## 📌 4️⃣ Digər Əhəmiyyətli Class-lar
| Class                  | Funksiya                                                                         |
| :--------------------- | :------------------------------------------------------------------------------- |
| **Authentication**     | User-in authentication məlumatını saxlayır (principal, credentials, authorities) |
| **GrantedAuthority**   | User-in icazələri (role və ya authority)                                         |
| **UserDetails**        | DB-dən gələn user məlumatlarını saxlayır                                         |
| **UserDetailsService** | Username ilə DB-dən user məlumatlarını çıxaran service                           |

---

## 📌 5️⃣ Spring Security Context Lifecycle

- Request gələndə SecurityContextPersistenceFilter `SecurityContextHolder`-dan context-i oxuyur.
- Authentication filter-ləri authentication edəndə Authentication obyekti context-ə qoyur.
- Authorization zamanı bu context-dəki Authentication yoxlanılır.
- Request bitəndə SecurityContextPersistenceFilter context-i təmizləyir.

---

## 📌 6️⃣ SecurityContextHolder Strategiyası

`ThreadLocal-based context strategy` istifadə edir.
Alternativ olaraq:
- MODE_INHERITABLETHREADLOCAL
- MODE_GLOBAL
strategiyalarını seçə bilərsən.

---

## 📌 7️⃣ Çox istifadə olunan Filter-lər

| Filter                                 | İşlədiyi yer                     |
| :------------------------------------- | :------------------------------- |
| `SecurityContextPersistenceFilter`     | Başlanğıc və bitiş               |
| `UsernamePasswordAuthenticationFilter` | /login request-i                 |
| `BasicAuthenticationFilter`            | HTTP Basic Auth üçün             |
| `JwtAuthenticationFilter`              | JWT token-lə authentication üçün |
| `ExceptionTranslationFilter`           | Exception handling üçün          |
| `FilterSecurityInterceptor`            | Authorization üçün               |
| `LogoutFilter`                         | /logout request-i                |


---

## 📌 Nəticə

Spring Security:

- ✅ Servlet Filter əsaslıdır
- ✅ SecurityContext-i ThreadLocal-da saxlayır
- ✅ Authentication və Authorization fərqli mərhələlərdə olur
- ✅ FilterChainProxy → SecurityFilterChain → Filter axışı ilə işləyir
- ✅ AuthenticationManager və Provider-lər authentication edir
- ✅ AccessDecisionManager authorization edir
- ✅ ExceptionTranslationFilter error handling edir
- ✅ SecurityContextPersistenceFilter context lifecycle-ı idarə edir

---

## 📌 Servlet və Filter nədir?

Əslində Java EE dünyasında:

- **Servlet** → Web serverdə (Tomcat, Jetty və s.) işləyən server-side proqram komponentidir, HTTP request-ləri qəbul edib cavab qaytarmaq üçün.
- **Filter** → Servlet-lərdən əvvəl və ya sonra işləyən aralıq interceptor-dur. Request və response üzərində müdaxilə etmək üçün.

Yəni request gələndə ilk Filter-lərdən keçir, sonra Servlet-ə çatır.
Filter-lər:

- request gələndə əvvəl işləyir
- cavab gedəndə də sonuncu işləyə bilir

### 📌 Spring Security-də Servlet və Filter-lər necə işləyir?

#### 📌 1️⃣ DispatcherServlet nədir?

Spring MVC-də bütün HTTP request-lər DispatcherServlet-ə düşür. O isə controller-ləri çağırır.

#### 📌 2️⃣ Security Filter Chain nədir?

Spring Security bütün təhlükəsizlik tədbirlərini SecurityFilterChain deyilən bir sıra filter-lərlə təşkil edir.
Bu filter-lər Servlet container-ə (Tomcat və s.) qeyd olunur və hər request gələndə sıra ilə icra olunur.

## 📌 Security FilterChain Axını (Default)
Gəlin nümunə bir axın göstərim:

**Client → (Tomcat) → SecurityFilterChain → DispatcherServlet → Controller**

SecurityFilterChain içində:

- UsernamePasswordAuthenticationFilter
- BasicAuthenticationFilter
- CsrfFilter
- ExceptionTranslationFilter
- SecurityContextPersistenceFilter
  və s. (bunların sırası vacibdir və öz order-i var)

### 📌 Tam axın: Client → Server → Servlet Container → Backend → Response

#### 📌 1️⃣ Client request göndərir
Client (browser, Postman və s.) HTTP protokolu ilə serverə sorğu göndərir.

Məsələn:
`GET http://localhost:8080/users`

#### 📌 2️⃣ Servlet Container nədir?
Servlet Container (məsələn Tomcat, Jetty, Undertow) Java web tətbiqlərini işlədən moduldu.

Vəzifəsi:
- Client-dən gələn HTTP sorğusunu qəbul etmək
- Onu `ServletRequest` obyektinə çevirmək
- Daxili filter və servlet-lərə ötürmək
- Cavabı `ServletResponse` obyektinə yığıb client-ə göndərmək

#### 📌 3️⃣ ServletRequest və ServletResponse
- **ServletRequest** → Client-in göndərdiyi sorğunun məlumatlarını saxlayır (URL, header, parametrlər, body və s.)
- **ServletResponse** → Serverin client-ə göndərəcəyi cavabın məlumatlarını saxlayır (status code, header, body və s.)

####  4️⃣ Filter-lər işə düşür
Servlet Container əvvəlcə öz qeyd olunan Filter-lər siyahısını (Filter Chain) yoxlayır.
Filter-lər:

- Request gələndə öncə işləyir.
- Response göndəriləndə sonda işləyə bilər.

Spring Security burada öz filter-lərini qeyd edir.
Məsələn:

- Auth yoxlanışı
- CSRF
- Exception handler
- Session yoxlanışı və s.

#### 📌 5️⃣ DispatcherServlet işə düşür
Request filter-lərdən keçəndən sonra Spring MVC-nin DispatcherServlet-inə çatır.

DispatcherServlet:

- Request-i qəbul edir
- Hansı controller və method-un bu request-i qarşılayacağını müəyyən edir (routing)
- Controller method-u çağırır

#### 📌 6️⃣ Controller işini görür
Controller:

- Request parametrlərini götürür
- Servis və ya repo çağırır
- İşini görüb ResponseEntity və ya başqa response qaytarır

#### 📌 7️⃣ Cavab DispatcherServlet-ə qayıdır
Controller-dən çıxan cavab:

- `DispatcherServlet`-ə qayıdır
- Oradan da Filter Chain-ə ötürülür (əgər response-də müdaxilə edən filter varsa, burada işə düşər)

#### 📌 8️⃣ ServletResponse client-ə göndərilir
Ən sonda Servlet Container:

- Response obyektini götürüb HTTP cavab halında client-ə qaytarır

---

## 🌟 Spring Security Filtrləri
Spring Security filtrləri hər bir HTTP sorğunu qarşılayaraq yoxlayır ki, həmin sorğu üçün autentifikasiya (doğrulama) tələb olunur, ya yox. Əgər autentifikasiya tələb olunursa, istifadəçi login səhifəsinə yönləndirilir və ya əvvəlki autentifikasiya zamanı yadda saxlanılmış məlumatlardan istifadə edilir.

## 🌟 Autentifikasiya (Authentication)
Məsələn, UsernamePasswordAuthenticationFilter kimi filtrlər HTTP sorğusundan istifadəçi adı və şifrəni çıxararaq Authentication tipində obyekt yaradır. Bu obyekt Spring Security-də autentifikasiya olunmuş istifadəçi məlumatlarının əsasını təşkil edir.

## 🌟 Autentifikasiya Meneceri (AuthenticationManager)
Filtrdən gələn sorğunu qəbul etdikdən sonra, istifadəçi məlumatlarının doğrulanmasını uyğun autentifikasiya təminatçılarına (AuthenticationProvider) yönləndirir. Bir tətbiqdə bir neçə təminatçı ola bilər və AuthenticationManager onların idarəsini üzərinə götürür. Sadə dillə desək, bu komponent autentifikasiyaya cavabdehdir.

## 🌟 Autentifikasiya Təminatçısı (AuthenticationProvider)
AuthenticationProvider autentifikasiya üçün lazım olan əsas doğrulama məntiqini özündə birləşdirir.

## 🌟 İstifadəçi Məlumatları Meneceri/Xidməti (UserDetailsManager / UserDetailsService)
Bu komponentlər istifadəçi məlumatlarını verilənlər bazasından və ya digər yaddaş sistemlərindən əldə etmək, yeniləmək və ya silmək üçün istifadə olunur.

## 🌟 Şifrə Kodlayıcı (PasswordEncoder)
Bu xidmət interfeysi şifrələrin kodlaşdırılması və həşlənməsi üçün istifadə olunur. Əks halda, parollar düz mətn (plain text) şəklində saxlanmalı olar ki, bu da təhlükəlidir ☹️

## 🌟 Təhlükəsizlik Konteksti (SecurityContext)
Sorğu uğurla autentifikasiya olunduqdan sonra, istifadəçiyə aid məlumatlar SecurityContext adlı lokal mühitdə saxlanılır. Bu mühit SecurityContextHolder tərəfindən idarə olunur və həmin istifadəçi ilə əlaqəli sonrakı sorğularda istifadə olunur.

---

## 📌 SessionCreationPolicy Enum Değerleri
- `STATELESS` -> Hiç session oluşturulmaz (JWT için ideal).
- `ALWAYS`	-> Her zaman session oluşturur (varsa bile yenisi yaratılır).
- `IF_REQUIRED` ->	Sadece gerekirse session oluşturur (default davranış).
- `NEVER` ->	Session oluşturmaz, ancak zaten varsa kullanır.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 👈 Oturum tutma!
            )
            .csrf(csrf -> csrf.disable()) // CSRF'yı kapat
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(OAuth2ResourceServerConfigurer::jwt); // JWT kullanımı

        return http.build();
    }
}
```
---

![Screenshot 2025-05-25 184328](https://github.com/user-attachments/assets/b5892bb1-0f3e-4d96-8cac-97765eba9941)

---

![security_image2](https://github.com/user-attachments/assets/de6650c8-ac3c-4c96-b742-f58ecd972ad0)

---

![security5](https://github.com/user-attachments/assets/7210a145-e653-4108-b609-c9b0ddbd753f)


---

![security_image3](https://github.com/user-attachments/assets/88541bcd-3465-4d32-97ae-3e97a8ad4269)

---

![Screenshot 2025-06-03 152914](https://github.com/user-attachments/assets/b435c992-9e8c-4d67-8050-10b5103a1b56)

