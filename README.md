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



