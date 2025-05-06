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

---

## 🧠 Qeydlər (Proxy və Mexanizmlər)

- Spring `TransactionInterceptor` ilə annotasiyaları oxuyur
- `PlatformTransactionManager` və `TransactionAttribute` obyektləri istifadə olunur
- AOP ilə proxy yaratdığı üçün `@Transactional` yalnız public metodlarda işləyir
- Eyni sinif daxilində self-invocation zamanı AOP işləməz

---

## ✅ Nəticə

Spring `@Transactional` annotasiyası ilə siz əməliyyatlarınızın **etibarlı və tam icrasını** təmin edə bilərsiniz. Parametrlərdən düzgün istifadə etməklə daha stabil, performanslı və təhlükəsiz sistem qura bilərsiniz.

