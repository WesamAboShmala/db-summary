
---

# 🚀 RMAN (Recovery Manager) & Disaster Recovery

---

## 1) ليش RMAN هي الأداة الحقيقية للـ Backup؟

**RMAN (Recovery Manager)** هي الأداة الحقيقية لعمل **Database Backup** كامل في أوراكل، وهي مختلفة تمامًا عن `exp`/`imp` (اللي بتاخد الـ Data بس).

> 💎 **RMAN بتاخد كل شي:**
> `Control File` + `SPFile` + `Data Files` + `Archive Log`

وبالتالي بتقدر ترجّع القاعدة بالكامل من الصفر بعد أي كارثة (**Disaster Recovery**)، بدون ما تحتاج تعيد ضبط أي إعدادات يدويًا.

---

## 2) الفكرة الجوهرية للـ Disaster Recovery 
لو ضربت القاعدة كلها (سيرفر جديد، جهاز جديد، ما في أي شي موجود)، بتقدري ترجّعيها بالكامل بس لو معك مسبقًا **3 أشياء أساسية**:

```text
┌──────────────────────────────────────────────────────────┐
│ 1. نسخة Backup (SPFile + Control File + DB + Archive Log) │
│ 2. رقم الـ DBID (لازم يكون محفوظ مسبقًا!)                  │
│ 3. برنامج RMAN وتطبيق الأوامر بالترتيب الصحيح             │
└──────────────────────────────────────────────────────────┘

```

---

## 3) أولاً: أخذ الباك أب (قبل أي كارثة، وأنتِ شغالة عادي) 🛡️

### 1️⃣ الاتصال وحفظ الـ DBID:

افتحي الـ Terminal وشغلي الأمر:

```bash
rman target /

```

> 📌 **تنبيه حرج:** سيرفعلك رقم الـ **DBID** احفظيه فورًا بملف نصي بمكان آمن خارج السيرفر!
---

### 2️⃣ تنفيذ أمر الباك أب الكامل:

```sql
run {
  backup spfile;
  backup current controlfile;
  backup archivelog all;
  backup database;
};

```

---

### 3️⃣ تفعيل الباك أب التلقائي للـ Control File:

```sql
CONFIGURE CONTROLFILE AUTOBACKUP ON;

```

---

## 4) ثانيًا: استرجاع القاعدة بالكامل (بعد الكارثة، على جهاز/سيرفر جديد) 🔄

| # | الخطوة | الأمر (Command) | شو بتعمل؟ (Explanation) |
| --- | --- | --- | --- |
| **1** | إنشاء Instance | `oradim -new -sid ucas` | تنشئ Instance فاضي بالاسم اللي بدك ياه |
| **2** | ملف المؤقت | إنشاء `init.ora` محتواه: `*.db_name='ucas'` | ملف Parameter مؤقت بسيط للتشغيل الأولي |
| **3** | الاتصال | `rman target /` | تتصلي بـ RMAN |
| **4** | تحديد الـ DBID | `set DBID=[رقم_DBID_المحفوظ]` | **خطوة حرجة جدًا، بدونها ما بتكملي!** |
| **5** | تشغيل مبدئي | `startup nomount pfile='D:\init.ora'` | تشغيل مبدئي بالملف البسيط |
| **6** | استرجاع Control | `restore controlfile;` | ترجعي الـ Control File |
| **7** | استرجاع SPFile | `restore spfile;` | ترجعي الـ SPFile الحقيقي |
| **8** | إعادة التشغيل | `shutdown immediate;` ثم `startup nomount;` | تعيدي التشغيل بالـ SPFile الحقيقي المسترجَع |
| **9** | ربط الفايلات | `alter database mount;` | تفتحي الـ Control File (Mount State) |
| **10** | استرجاع البيانات | `restore database;` | ترجعي ملفات البيانات الفعلية (بتاخذ وقت دقايق) |
| **11** | تطبيق التغييرات | `recover database;` | تطبّقي الـ Archive Log والـ Redo Log (Instance Recovery) |
| **12** | الفتح النهائي | `alter database open resetlogs;` | فتح نهائي + إعادة ترقيم الـ SCN |

---

## 5) التحقق من نجاح العملية ✅

بعد الفتح النهائي، تأكدي من استرجاع جدول بياناتك كاملاً:

```sql
SELECT * FROM tab;

```

---

## 🧠 الخلاصة والشي الوحيد اللي لازم تكوني فاهمتيه

```text
📥 الباك أب:
rman target / ➔ backup database (+ spfile + controlfile + archivelog)

📤 الاسترجاع (Recovery):
set DBID ➔ startup nomount ➔ restore controlfile/spfile ➔ mount ➔ restore database ➔ recover database ➔ open resetlogs

```

> 💡 **ملاحظة:**
> كل شي ثاني (الـ Catalog، الـ Retention Policy، الـ Backup Set مقابل Image Copy، الـ Data Guard) هو تفاصيل وخيارات إضافية مش أساس السيناريو الرئيسي.

---
# 🕊️ ختاماً

> *"المهندس الحقيقي لا ينتظر وقوع الكارثة ليتعلم كيف يواجهها، بل يبني النظام الذي يحمي البيانات قبل أن تبدأ."*

**تَمَّ بِحَمْدِ اللَّهِ وَتَوْفِيقِهِ إِتْمَامُ دَلِيلِ إِدَارَةِ وَأَمْنِ قَوَاعِدِ البَيَانَاتِ (Oracle Database Security).**

نسأل الله أن يجعل هذا العلم نافعاً ومباركاً، وأن يكون خطوةً نحو التميز والتوفيق في المسيرة الأكاديمية والمهنية.

✨ **اللهم صلِّ وسلِّم وبارك على سيدنا ونبينا محمد وعلى آله وصحبه أجمعين.**

🤲 **نسألكم خالص الدعاء بظهر الغيب.**
