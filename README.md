# Transactional-Deep-Dive
# 🔄 `@Transactional` Annotasiyası – Dərin İzah (Spring Framework)

`@Transactional` annotasiyası Spring Framework-də **verilənlər bazası əməliyyatlarını** (CRUD əməliyyatları) **transaksiya** (transaction) kimi idarə etmək üçün istifadə olunur. Məqsəd: ya bütün əməliyyatların **tam uğurla** başa çatması (`commit`), ya da hər hansı bir problem baş verərsə **hər şeyin geri qaytarılmasıdır** (`rollback`).

---

## 📌 Transaksiya nədir?

Verilənlər bazasında bir neçə əməliyyat bir arada yerinə yetirilirsə və bu əməliyyatlar ya **tam uğurla** həyata keçirilməli, ya da **heç biri** icra olunmamalıdırsa — bu cür əməliyyata "transaksiya" deyilir.

### ✅ ACID Prinsipləri
1. **Atomicity** – əməliyyatlar bölünməzdir.
2. **Consistency** – bazanın integrasiyası qorunur.
3. **Isolation** – bir neçə transaksiya bir-birindən asılı olmadan işləyir.
4. **Durability** – commit olunduqdan sonra məlumat qalıcı olur.

---

## 🧠 `@Transactional` necə işləyir?

Spring, `@Transactional` annotasiyasını AOP (Aspect-Oriented Programming) ilə **proxy** yaradaraq həyata keçirir. Yəni:

- Əgər class **interface**-dən istifadə edirsə → JDK Dynamic Proxy
- Əgər yoxdursa → CGLIB ilə class-ın subclass-ı yaradılır

Metod çağırışı bu proxy üzərindən keçərək:

- Transaksiya başlayır (`begin`)
- Əgər metod uğurla bitərsə → `commit`
- Əgər exception baş verərsə → `rollback`

---

## 📌 Anotasiyanın Parametrləri

| Parametr | İzah |
|----------|------|
| `propagation` | Transaksiyanın hansı şəkildə idarə olunacağını göstərir |
| `isolation` | Transaksiyalar bir-birindən nə qədər təcrid olunacaq |
| `readOnly` | Yalnız oxuma əməliyyatları üçün optimizasiya |
| `rollbackFor` | Hansı exception-lar üçün rollback olacaq |
| `noRollbackFor` | Hansı exception-lar rollback olmayacaq |
| `timeout` | Transaksiya neçə saniyəyə qədər aktiv qala bilər |

---

# 🔁 `propagation` növlərinin DƏRİN izahı və nümunələri

## 1. `REQUIRED` (default)

### Açıklama:
- Əgər çağıran metodda artıq transaksiya varsa → **o transaksiyanı istifadə edir**
- Əgər yoxdursa → **yeni transaksiya yaradır**

### Davranış:
- Mövcud transaksiyaya qoşulur
- Əgər child metod exception atarsa → **bütün transaksiya rollback olur**

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    @Transactional
    public void methodA() {
        serviceB.methodB(); // methodB eyni transaksiya içindədir
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.REQUIRED)
    public void methodB() {
        // Mövcud transaksiyaya qoşulur
    }
}
```

---

## 2. `REQUIRES_NEW`

### Açıklama:
- Hər zaman **yeni transaksiya** yaradılır
- Əgər mövcud transaksiya varsa → onu **suspend** (dayandırır)

### Davranış:
- İstisna baş versə də parent rollback olmur

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    @Transactional
    public void methodA() {
        serviceB.methodB(); // ayrı transaksiya
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // Yeni transaksiya → methodA-dan asılı deyil
    }
}
```

---

## 3. `NESTED`

### Açıklama:
- Əgər mövcud transaksiya varsa → **nested savepoint** yaradılır
- Əgər yoxdursa → yeni transaksiya yaradılır

### Davranış:
- Child metod exception atarsa → yalnız o hissə rollback olur, parent davam edir

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    @Transactional
    public void methodA() {
        try {
            serviceB.methodB(); // nested
        } catch (Exception e) {
            // yalnız methodB rollback olacaq
        }
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.NESTED)
    public void methodB() {
        // Nested savepoint yaradılır
    }
}
```

---

## 4. `MANDATORY`

### Açıklama:
- **Mövcud transaksiya olmalıdır**
- Əgər yoxdursa → `IllegalTransactionStateException` atır

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    public void methodA() {
        serviceB.methodB(); // exception atılacaq
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.MANDATORY)
    public void methodB() {
        // Mövcud transaksiya olmadığından exception
    }
}
```

---

## 5. `NEVER`

### Açıklama:
- Əgər transaksiya varsa → **exception atır**
- Yalnız transaksiyasız işləməyə icazə verir

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    @Transactional
    public void methodA() {
        serviceB.methodB(); // exception atılacaq
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.NEVER)
    public void methodB() {
        // Transaksiya varsa işləməz
    }
}
```

---

## 6. `SUPPORTS`

### Açıklama:
- Əgər transaksiya varsa → istifadə edir
- Yoxdursa → transaksiyasız işləyir

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    public void methodA() {
        serviceB.methodB(); // transaksiyasız işləyir
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.SUPPORTS)
    public void methodB() {
        // vəziyyətə uyğun işləyir
    }
}
```

---

## 7. `NOT_SUPPORTED`

### Açıklama:
- Əgər transaksiya varsa → **suspend edir**, sonra transaksiyasız işləyir

### Kod nümunəsi:

```java
@Service
public class ServiceA {
    @Transactional
    public void methodA() {
        serviceB.methodB(); // transaksiyasız işləyəcək
    }
}

@Service
public class ServiceB {
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void methodB() {
        // Mövcud transaksiya dayandırılır
    }
}
```

## Spring Framework-də `@Transactional` annotasiyası verilənlər bazası əməliyyatlarının təhlükəsiz və idarəolunan şəkildə icra olunması üçün istifadə olunur. Bu annotasiyanın arxasında bir çox komponent və mexanizm gizlidir. Bu sənəddə onları dərin izah edirik:

### 🧠 1. `TransactionInterceptor` – Əsas Mexanizm

### Nədir?
Spring AOP `@Transactional` annotasiyasını işlətmək üçün `TransactionInterceptor` adlı bir interceptor sinifindən istifadə edir.

### Necə işləyir?

1. Metoda çağırış gəlir (məsələn `paymentService.pay()`).
2. Əgər bu metod `@Transactional` annotasiyası ilə işarələnibsə, Spring həmin metodu **proxy vasitəsilə** yönləndirir.
3. `TransactionInterceptor`:
   - Annotasiyanı oxuyur
   - `TransactionAttribute` obyektini yaradır
   - `PlatformTransactionManager` ilə yeni/mövcud transaksiya əldə edir
   - Əsas metodu çağırır
   - Əgər uğurludursa: `commit`, əks halda: `rollback`

### Koddan parçalar (sadələşdirilmiş məntiq):

```java
TransactionAttribute attr = transactionAttributeSource.getTransactionAttribute(method, targetClass);
TransactionStatus status = transactionManager.getTransaction(attr);

try {
    // Əsas metodun icrası
    Object result = methodInvocation.proceed();
    transactionManager.commit(status);
    return result;
} catch (Throwable ex) {
    transactionManager.rollback(status);
    throw ex;
}
```

### 🧾 2. TransactionAttribute – Annotasiyanın Parametrləri
- Spring bu obyekt vasitəsilə @Transactional annotasiyasının bütün parametrlərini saxlayır:
    - propagation
    - isolation
    - readOnly
    - rollbackFor
    - timeout

- Bu parametrlər metod üçün tranzaksiya davranışını təyin edir.


### 🔧 3. PlatformTransactionManager – Tranzaksiyanı İdarə Edən Mexanizm
- Spring-in tranzaksiya idarə edən interfeysidir. Müxtəlif verilənlər bazası texnologiyalarına uyğun implementasiyaları var:

| İstifadə sahəsi | İmplementasiya |
|----------------|----------------|
| JDBC           | `DataSourceTransactionManager` |
| JPA / Hibernate| `JpaTransactionManager` |
| JTA            | `JtaTransactionManager` |

```java
@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}
```

### 🪞 4. Proxy Mexanizmi
- @Transactional sadəcə metod səviyyəsində deyil, proxy üzərindən işləyir.

Proxy növləri:

| Şərt | İstifadə olunan proxy |
|------|------------------------|
| Sinif `interface`-dən implement edirsə | JDK Dynamic Proxy |
| Yoxdursa | CGLIB Proxy (sinif subclass olunur) |

Misal:

```java
@Service
public class MyService {
    @Transactional
    public void performAction() {
        // Bu metod bir proxy class vasitəsilə çağırılacaq
    }
}

```
Proxy-lər vasitəsilə Spring metod çağırışlarının qarşısında TransactionInterceptor yerləşdirir.

### 🚫 5. Self-Invocation Problemi (Daxili Çağırışlar)

Problem:
- Bir sinifin içində bir @Transactional metod başqa @Transactional metodu çağırarsa — tranzaksiya işləməz.

Səbəb:
- Çağırış proxy vasitəsilə deyil, birbaşa obyektin özü ilə olur.

Problemlə qarşılaşan kod:

```java
@Service
public class AccountService {

    @Transactional
    public void transfer() {
        withdraw(); // <--- Tranzaksiya başlamayacaq!
    }

    @Transactional
    public void withdraw() {
        // bu metod heç bir tranzaksiya ilə icra olunmayacaq
    }
}
```
Həll 1: Metodu başqa servisdə yaz:

```java
@Service
public class TransferService {
    @Autowired
    private WithdrawService withdrawService;

    @Transactional
    public void transfer() {
        withdrawService.withdraw(); // <--- proxy vasitəsilə çağırılır
    }
}

@Service
public class WithdrawService {
    @Transactional
    public void withdraw() {
        // indi tranzaksiya işləyir
    }
}
```

Həll 2: ApplicationContext ilə öz proxy-nu çağır:

```java
@Service
public class AccountService {

    @Autowired
    private ApplicationContext context;

    @Transactional
    public void transfer() {
        context.getBean(AccountService.class).withdraw(); // Proxy vasitəsilə çağırılır
    }

    @Transactional
    public void withdraw() {
        // tranzaksiya indi işləyir
    }
}
```

✅ Nəticə

| Mexanizm | İzah |
|----------|------|
| `TransactionInterceptor` | Annotasiyaları oxuyur və tranzaksiya idarəsini təmin edir |
| `PlatformTransactionManager` | Əsas transaksiya əməliyyatlarını yerinə yetirir |
| `TransactionAttribute` | Annotasiya parametrlərini saxlayır |
| Proxy | `@Transactional` yalnız **proxy vasitəsilə** işləyir |
| Self-Invocation | Eyni sinif daxilində çağırış zamanı tranzaksiya işləməz |

---

## 📌 @Transactional Annotation ile save() metodu olmadan save etmək

```java
@Transactional
public void updateUser(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName);
    // userRepository.save(user); ← bunu çağırmırıq!
}
```

- Cavab: JPA'nın Persistence Context (Qalıcı Obyekt Hovuzu) və Dirty Checking mexanizması sayəsində save olur.

### 📌 Persistence Context Nedir?
- `Persistence Context` bütün əməliyyat boyunca idarə olunan obyektlərin keşi kimi işləyir.
    - Siz `findById` ilə obyekti əldə etdiyiniz zaman bu obyekt `Persistence Context`’e (idarə olunan vəziyyət(`managed state`)) götürülür.
    - Bu obyektdə təyinetmə metodları ilə edilən dəyişikliklər `Persistence Context` deki snapshot ilə müqayisə edilir.
 

### 📌 Dirty Checking (Kirli Kontrol) Mekanizması

- Transaksiya bağlanarkən (metod bitərkən):
    - Persistence Context, transaksiya ərzində dəyişən entity-ləri yoxlayır.
    - Əgər bir entity-nin sahələrində dəyişiklik edilibsə (snapshot ilə müqayisə edərək), bunları SQL UPDATE ifadəsi kimi verilənlər bazasına ötürür.
    - Bu prosesə Dirty Checking deyilir.
    - Yəni save() çağırmağa ehtiyac yoxdur.
 
### 📌 Nə vaxt işləyir?
- Yalnız @Transactional annotasiyası altında.
- Yalnız managed state entity-lərdə (idarə olunan vəziyyətdə olan entity-lərdə).
- Transaction commit edilərkən işləyir.

### 📌 Prosesin Axışı

p- Transaction başlayır
- `findById` → entity Persistence Context-ə əlavə olunur
- `setName()` → entity-nin sahəsi dəyişir
- `Transaction` `commit` edilərkən:
- `Persistence Context`, entity-nin snapshot-ını hazırki vəziyyəti ilə müqayisə edir
- Dəyişiklik varsa → SQL UPDATE sorğusu işlədir
- Dəyişiklik yoxdursa → heç nə etmir
- (Yəni, avtomatik olaraq dəyişikliklər bazaya yazılır, əl ilə `save()` çağırmağa ehtiyac yoxdur.)

`Dirty Checking`, `JPA/Hibernate` tərəfindən avtomatik olaraq entity dəyişikliklərini aşkarlayıb bazaya yazma mexanizmidir. Yalnız transaction daxilində və managed entity-lər üçün işləyir.

---

## 🧠 Qeydlər (Proxy və Mexanizmlər)

- Spring `TransactionInterceptor` ilə annotasiyaları oxuyur
- `PlatformTransactionManager` və `TransactionAttribute` obyektləri istifadə olunur
- AOP ilə proxy yaratdığı üçün `@Transactional` yalnız public metodlarda işləyir
- Eyni sinif daxilində self-invocation zamanı AOP işləməz
---

## ✅ Nəticə

Spring `@Transactional` annotasiyası ilə siz əməliyyatlarınızın **etibarlı və tam icrasını** təmin edə bilərsiniz. Parametrlərdən düzgün istifadə etməklə daha stabil, performanslı və təhlükəsiz sistem qura bilərsiniz.

