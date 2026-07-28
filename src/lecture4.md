
بسم الله الرحمن الرحيم
> 🤲 **"اللهم انفعنا بما علمتنا، وعلمنا ما ينفعنا، وزدنا علماً"**

# المراقبة المتقدمة لبيانات أوراكل (Extended Auditing, Triggers, FGA & DBA Auditing)

<div style="background-color: #f0f7ff; border-right: 5px solid #2563eb; padding: 15px 20px; border-radius: 8px; margin: 20px 0; color: #1e293b;">
  <p style="margin: 0; font-weight: bold; font-size: 1.05em;">🤲 تذكير ودعاء:</p>
  <p style="margin: 8px 0 0 0; font-style: italic; color: #334155;">"اللهم لا سهل إلا ما جعلته سهلاً، وأنت تجعل الحزن إذا شئت سهلاً.. صلي على النبي ﷺ وخد نفس عميق وابدأ معي!"</p>
</div>

## 1. كيف نعرف القيمة القديمة والجديدة عند التعديل (UPDATE)؟

### المشكلة الأساسية:

الأوديت العادي (**Basic Audit**) بس بيقولك: *"فلان عمل UPDATE على الجدول الفلاني بالوقت الفلاني"*.
هاد الكلام مش كافي كـ DBA! لو صار خطأ أو احتيال، أنت محتاج تعرف: **شو كانت القيمة قبل التعديل (:OLD) وشو صارت بعده (:NEW)؟**

> ⚠️ **ملاحظة مهمة:**
> عمليات الـ `SELECT` من أخطر الصلاحيات بالداتابيز، لأن المستخدم بيقدر ينسخ الجدول كامل (`CREATE TABLE AS SELECT *`)؛ عشان هيك لازم يكون عليها Audit زيها زي الـ `INSERT/UPDATE/DELETE`.

---

### الطريقة الأولى: Database Extended Auditing

#### شو هي؟

إعداد بسيط بتفعله بقاعدة البيانات عن طريق الـ Parameter:

```sql
AUDIT_TRAIL = DB, EXTENDED

```

#### شو بتعمل؟

أوراكل بتصير تحفظ نص جملة الـ SQL كاملة اللي تنفذت داخل الـ Audit Trail.
*مثال:* لو يوزر كتب: `UPDATE test SET id = 55 WHERE id = 10;` النص هاد كامل بينحفظ.

#### عيوب هالطريقة (ليش مش مثالية؟):

1. **حجم تخزين ضخم (Storage & Performance):** مع آلاف العمليات يومياً رح يتخزن كم هائل من النصوص، وهاد بيضرب مبدأ الـ Normalization وبيبطئ الأداء.
2. **صعوبة القراءة:** بدل ما تلاقي جدول منظم (قديم/جديد)، رح تلاقي كومة جمل SQL خام محتاجة تحليل يدوي.
3. **خطر منطقي عند الصيانة:** لو أوقفت الأوديت لفترة صيانة وصار تعديل، لما ترجع تشغله رح تفوتك المعلومة وتظن إن آخر قيمة مسجلة هي الصح.

---

### الطريقة الثانية: استخدام الـ Trigger (الأذكى والأكثر عملية)

الـ **Trigger** هو حارس مبرمج بيصحى وينفذ أوتوماتيكياً لما يصير حدث معين (مثل `UPDATE`).

#### الخطوة 1: إنشاء جدول مخصص لسجل التغييرات (`Audit Log Table`)

```sql
CREATE TABLE test_tb_audit (
    old_id      NUMBER,
    new_id      NUMBER,
    action_time DATE,
    action      VARCHAR2(20),
    user_name   VARCHAR2(50)
);

```

#### الخطوة 2: إنشاء الـ Trigger

```sql
CREATE OR REPLACE TRIGGER test_tb_audit_trg
AFTER UPDATE ON test_tb
FOR EACH ROW
BEGIN
  INSERT INTO test_tb_audit (old_id, new_id, action_time, action, user_name)
  VALUES (:OLD.id, :NEW.id, SYSDATE, 'UPDATE', USER);
END;
/

```

#### 🔍 تفكيك الكود خطوة بخطوة:

* **`CREATE OR REPLACE TRIGGER`**: ما في أمر اسمه `ALTER TRIGGER` لتعديل الكود، فبنستخدم `CREATE OR REPLACE`.
* **`AFTER UPDATE ON test_tb`**: بيتنفذ بعد ما تتم عملية التعديل على جدول `test_tb`.
* **`FOR EACH ROW`**: **(أهم جزئية!)** لو التعديل أثر على 7 صفوف، الـ Trigger بينفذ 7 مرات ليعطيك تفاصيل كل صف. لو شلتها رح ينفذ مرة وحدة بس وتضيع عليك تفاصيل باقي الصفوف!
* **`:OLD.id` و `:NEW.id**`: كلمات محجوزة بـ PL/SQL. `:OLD` القيمة قبل التعديل، و `:NEW` القيمة بعد التعديل.
> 💡 **تنبيه من المحاضرة:** انتبه ما تعكس `:OLD` مكان `:NEW` بالغلط وأنت بتكتب الـ Insert حتى ما تطلع النتائج معكوسة!


* **`SYSDATE` & `USER**`: دالّات جاهزة لترجيع الوقت الحالي واسم اليوزر اللي نفذ الحركة.

---

### 📊 مقارنة سريعة بين الطريقتين

| وجه المقارنة | Extended Audit | Trigger |
| --- | --- | --- |
| **شو بيخزن؟** | نص جملة الـ SQL كاملة | بس الأعمدة المهمة (قديم / جديد) |
| **سهولة القراءة** | صعبة (محتاجة تحليل نصوص) | سهلة جداً (جدول مرتب) |
| **تأثير المساحة** | يستهلك مساحة كبيرة جداً | خفيف ومُنظّم |
| **الجهد بالإعداد** | أسهل (تفعيل Parameter) | محتاج كتابة كود وبرمجة |

---

## 2. المراقبة الدقيقة (Fine-Grained Auditing - FGA)

بدل ما نراقب كل العمليات على الجدول ونغرق بالمعلومات، الـ **FGA** بتخلينا نراقب **فقط العمليات اللي بتحقق شرط معين!**

> 💡 **مثال حقيقي:**
> جدول الرواتب فيه آلاف عمليات الاستعلام (`SELECT`) يومياً. ما بيهمك الاستعلام العادي، بس بيهمك تطلع Alert لو حدا استعلم عن **رواتب الإدارة العليا** (مثلاً: `WHERE salary > 10000`).

### ليش بنستخدم FGA؟

1. **تسهيل قراءة السجلات:** نوصل للمعلومة المهمة فوراً.
2. **تقليل حجم التخزين:** مساحة أقل بكثير على الداتابيز.
3. **التحقق من استخدام الصلاحيات:** التأكد من إن اليوزر بيستخدم صلاحياته صح.
4. **التنبيهات الفورية (Notifications):** إمكانية إرسال إيميل أو SMS فور تحقق الشرط.

> ⚖️ **قاعدة ذهبية (مبدأ التوازن):**
> *"كل ما ريحت حالك كمبرمج، كل ما تعبت الداتابيز.. والعكس صحيح!"*
> لا تستخدم FGA على كل شيء حتى ما تبطئ النظام. اعمل **Data Classification** وراقب بس البيانات الحساسة (**Sensitive Data**).

---

### التطبيق العملي للـ FGA

#### 1️⃣ الطريقة البسيطة: استخدام `DBMS_FGA`

أوراكل بتوفر Package جاهزة لإضافة السياسات:

```sql
-- إضافة السياسة (Add Policy)
BEGIN
 DBMS_FGA.ADD_POLICY(
   object_schema   => 'HR',
   object_name     => 'test_tb',
   policy_name     => 'ID_AUDIT',
   audit_condition => 'id > 100',
   statement_types => 'SELECT'
 );
END;
/

```

* **التجربة:** لو يوزر عمل `SELECT * FROM test_tb WHERE id = 10` **ما رح يتسجل شي**. بس لو عمل `WHERE id = 107` **رح يتسجل فوراً** بالـ Log الخاص بأوراكل (`DBA_FGA_AUDIT_TRAIL`).

```sql
-- إلغاء أو حذف السياسة (Drop Policy)
BEGIN
 DBMS_FGA.DROP_POLICY(
   object_schema => 'HR',
   object_name   => 'test_tb',
   policy_name   => 'ID_AUDIT'
 );
END;
/

```

---

#### 2️⃣ الطريقة المتقدمة: استخدام الـ Handler (إجراء إضافي)

الـ **Handler** يعني: *"لما يتحقق الشرط، مش بس تسجّل بالـ Log، نفّذ كمان كود إضافي (مثلاً ابعت إيميل أو خزن بجدول خاص)"*.

**الخطوات:**

1. إنشاء جدول مخصص لتلقي التنبيهات:

```sql
CREATE TABLE fga_audit_table (
    text             VARCHAR2(4000),
    transaction_date DATE,
    user_id          VARCHAR2(50)
);

```

2. إنشاء الـ Handler Procedure:

```sql
CREATE OR REPLACE PROCEDURE fga_handle (
  object_schema IN VARCHAR2,
  object_name   IN VARCHAR2,
  policy_name   IN VARCHAR2
) AS
  v_text VARCHAR2(4000);
BEGIN
  v_text := 'FGA Alert on ' || object_schema || '.' || object_name || ' using policy ' || policy_name;
  
  INSERT INTO fga_audit_table (text, transaction_date, user_id)
  VALUES (v_text, SYSDATE, USER);
END;
/

```

3. ربط الـ Handler بالـ Policy:

```sql
BEGIN
 DBMS_FGA.ADD_POLICY(
   object_schema   => 'HR',
   object_name     => 'test_tb',
   policy_name     => 'ID_AUDIT',
   audit_condition => 'id > 100',
   statement_types => 'SELECT',
   handler_schema  => 'HR',
   handler_module  => 'fga_handle'
 );
END;
/

```

* **النتيجة:** تسجيل مزدوج! بيتسجل أوتوماتيك بجدول أوراكل، وبنفس الوقت بيتنفذ الـ Procedure وبيخزن بنسختنا المخصصة.

---

## 3. مراقبة الـ DBA وضمان فصل الصلاحيات (Separation of Duties)

**السؤال الأهم:** لو الـ DBA (صاحب أصل الصلاحيات) هو اللي مسح البيانات أو تلاعب بالسجلات.. مين بيراقبه؟!

> 🛡️ **مبدأ فصل الصلاحيات:**
> الـ Developer يراقبه الـ DBA.. والـ DBA يراقبه نظام التشغيل (OS)! لا أحد فوق المراقبة.

### كيف بنفعّل مراقبة الـ DBA؟

عن طريق الأمر التالية (محتاج إعادة تشغيل للداتابيز):

```sql
ALTER SYSTEM SET AUDIT_SYS_OPERATIONS = TRUE SCOPE = SPFILE;

```

### أين تُخزن عمليات الـ DBA؟

* **عمليات المستخدم العادي:** بتتخزن داخل قاعدة البيانات نفسها.
* **عمليات الـ DBA / SYS:** بتتخزن **خارج قاعدة البيانات** بملف على مستوى نظام التشغيل (**Event Viewer** في Windows أو **Audit Trail** في Linux/Unix).

**ليش خارج الداتابيز؟** حتى لو الـ DBA حاول يحذف حركاته أو ينظف الـ Tables جوا الداتابيز، ما يقدر يمسح الملفات المكتوبة على مستوى الـ OS!

*مثال على حركة مريبة للـ DBA يتم تسجيلها في الـ Event Viewer:*

```sql
CREATE USER manager IDENTIFIED BY manager;
GRANT DBA TO manager;
DROP USER manager; -- أنشأ يوزر وأعطاه صلاحيات عالية ثم حذفه لإخفاء الآثار!

```

---

## 📌 الخلاصة الشاملة لمنظومة الـ Audit

1. **Failed Login Attempts:** مراقبة محاولات الدخول الفاشلة لحماية النظام من التخمين.
2. **Changes to Critical Tables:** مراقبة القيم القديمة والجديدة عند التعديل (`Extended Audit` أو `Trigger`).
3. **FGA:** المراقبة المشروطة والمركزة على استعلامات محددة وتقليل الضوضاء بالسجلات.
4. **Auditing DBA Actions:** مراقبة كبار المستخدِمين وتخزين سجلاتهم على مستوى نظام التشغيل لتطبيق مبدأ **Separation of Duties**.
