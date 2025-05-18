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
