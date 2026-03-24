# Impact — قياس الأثر (Pillar 01/02/03)


## نظرة عامة
- الهدف: توفير رؤية شاملة قابلة للتنفيذ عن أثر المنصّة على التعلم من ثلاث زوايا متكاملة:
  - Pillar 01: قياس رضا المستخدم (User Satisfaction Measurement)
  - Pillar 02: إدارة أداء الطلبة (Student Performance Management)
  - Pillar 03: جودة المحتوى الذكي وامتثاله (AI Content Quality & Alignment)
- مصادر البيانات الأساسية:
  - محاولات التقييم، عناصر الأسئلة، البطاقات التعليمية، الملخصات، سجلات التتبّع والوقت، والآراء والتقييمات والمشكلات من جداول feedback (المركزية) وservice_feedback (الخدمة).
- مبادئ الوصول والحماية:
  - معظم واجهات Impact محمية بالأدوار (Admin/Content Reviewer/Org Admin)، وتستخدم حداً للطلبات (Rate Limit) وتخزيناً مؤقتاً (Caching) حيث يلزم.


## البنية البياناتية سريعة
- جداول رئيسية: attempts، quiz_items، flashcards، unit_summary، content_feedback، service_feedback.
- ملاحظات:
  - feedback_analytics تعتمد أولاً على حقول denormalized في الجداول الأساسية، ثم تسقط إلى service_feedback كمسار احتياطي عندما تكون المؤشرات الأساسية غير محدثة.
  - تقييم جودة الملخصات/البطاقات يتم عبر LLM ويخزن في evaluation_details/quality_score حيثما ينطبق.


## Pillar 01 — قياس رضا المستخدم

**ماذا نقيس؟**
- متوسط التقييمات (1–5)
- عدد التقييمات
- عدد الإعجابات وعدم الإعجاب (عند التتبع)
- إجمالي المشكلات حسب النوع
- العناصر منخفضة التقييم (Low‑Rated) والعناصر عالية المشاكل (High‑Issue)

**واجهات برمجة التطبيقات**
- ملخص جودة أسئلة التقييم:
  - GET /api/v1/feedback/assessment/summary
  - معلمات مهمة: artifact_id، unit_id، max_rating، min_issue_count، min_ratings_count، limit
  - مرجع الكود: [feedback_analytics.py: get_question_quality_summary](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_analytics.py#L43-L251)
- ملخص جودة البطاقات التعليمية:
  - GET /api/v1/feedback/flashcards/summary
  - مرجع الكود: [feedback_analytics.py: get_flashcard_quality_summary](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_analytics.py#L252-L461)
- ملخص جودة ملخصات الوحدات:
  - GET /api/v1/feedback/unit-summary/summary
  - مرجع الكود: [feedback_analytics.py: get_unit_summary_quality_summary](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_analytics.py#L462-L751)

**نقاط تصميمية**
- fallback إلى جدول service_feedback عندما تكون الحقول المجملة غير متوفرة لضمان عدم ظهور أقسام فارغة.
- ضبط العتبات عبر المعلمات (max_rating، min_issue_count، min_ratings_count) لمرونة العرض في لوحة الإدارة.

**تجميع الآراء (Admin)**
- عرض شامل للآراء والمشكلات مع ترشيح وترقيم صفحات:
  - GET /api/v1/admin/feedback/metrics
  - مرجع: [feedback_admin_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_admin_api.py)


## Pillar 02 — إدارة أداء الطلبة

**الأهداف**
- متابعة الكفاءة والفاعلية التعليمية عبر الزمن (ROI، Effectiveness)
- تحليل فجوات المهارات (Skill Gaps) واتجاهات التحسن
- تفعيل أدوات IRT للتحقق من ملاءمة العناصر والمعايرة

**واجهات برمجة التطبيقات (Analytics Impact)**
- ROI للمستخدم الحالي:
  - GET /api/v1/analytics/impact/roi/{time_range}
  - مرجع: [analytics/api/impact.py: get_own_roi](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/api/impact.py#L75-L149)
- ملخص أداء الطالب (مجمّع):
  - GET /api/v1/analytics/impact/student/performance/summary
  - مرجع: [analytics/api/impact.py: get_student_performance_summary](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/api/impact.py#L151-L177)
- ROI/Effectiveness/Skill Gaps للمؤسسة:
  - GET /api/v1/analytics/impact/roi/organization/{org_id}/{time_range}
  - GET /api/v1/analytics/impact/effectiveness/organization/{org_id}/{time_range}
  - GET /api/v1/analytics/impact/skill-gaps/organization/{org_id}/{time_range}
  - نفس الملف: [analytics/api/impact.py](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/api/impact.py)

**IRT (تحليل العناصر واستقرار النموذج)**
- تقييمات IRT: تقارب النموذج، ملاءمة العناصر، المعايرة، مسارات theta
  - مسارات: /api/v1/irt/evaluation/*
  - مرجع: [irt_evaluation.py](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/irt_evaluation.py)

**ملاحظات تنفيذية**
- التخزين المؤقت يُستخدم في بعض مؤشرات ROI/Effectiveness (24 ساعة افتراضياً).
- التحكم بالصلاحيات: عرض فردي للمستخدم نفسه، وعرض المؤسسة للأعضاء/المديرين حسب سياسة الأذونات.


### معادلات وعلاقات Pillar 02
- استثمار الوقت:
  - تجميع زمن الدراسة والاختبارات:
  
    ```text
    total_learning_minutes = study_seconds/60 + quiz_seconds/60
    total_hours = total_learning_minutes / 60
    active_days = max(study_days, quiz_days)
    avg_study_session_minutes = (study_seconds/study_days)/60
    avg_quiz_session_minutes  = (quiz_seconds/quiz_attempts)/60
    ```
  
  - مرجع: [impact_metrics_service.py:_calculate_time_investment](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L678-L784)

- المهارة المكتسبة:
  - تحسين الدرجة عبر الفترة المختارة:
  
    ```text
    early_avg_pct  = average(%score of first 10 attempts in range)
    recent_avg_pct = average(%score of last 10 attempts in range)
    score_improvement = recent_avg_pct - early_avg_pct
    ```
  
  - الإتقان والإنهاء:
  
    ```text
    subjects_mastered = count(subject.score >= 70 and attempts >= 3)
    units_completed   = count(unit.score >= 50 and attempts >= 1)
    ```
  
  - مرجع: [impact_metrics_service.py:_calculate_skill_gained](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L786-L829), [impact_metrics_service.py:_calculate_score_improvement](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L830-L875)

- ROI:
  - التعريف العام:
  
    ```text
    ROI ∝ Skill Gained / Time Invested
    ```
  
  - التطبيق الفعلي في الخدمة:
  
    ```text
    if total_hours > 0 and score_improvement is not None:
        roi_score = (score_improvement * 10) / total_hours
    elif total_hours > 0:
        roi_score = (average_score * 5) / total_hours
    else:
        roi_score = 0
    ```
  
  - تصنيف العتبات (Benchmark):
  
    ```text
    if total_hours < 1           -> "insufficient_data"
    elif roi_score >= 50         -> "excellent"
    elif roi_score >= 30         -> "good"
    elif roi_score >= 15         -> "average"
    elif roi_score >= 5          -> "below_average"
    else                         -> "poor"
    ```
  
  - معدل اكتساب المهارة:
  
    ```text
    acquisition_rate = subjects_mastered / total_hours   # subjects/hour
    ```
  
  - مرجع: [impact_metrics_service.py:calculate_roi](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L98-L176), [impact_metrics_service.py:_calculate_roi_benchmark](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L876-L896), [impact_metrics_service.py:_calculate_acquisition_rate](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L911-L930)

- سرعة التعلم (Velocity):
  
  ```text
  attempts_per_day = total_attempts / days_in_range
  velocity_score   = min(100, attempts_per_day * 20)   # 5 attempts/day -> 100
  ```
  
  - مرجع: [impact_metrics_service.py:_calculate_velocity](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L995-L1020)

- الاحتفاظ بالمعرفة (Retention):
  
  ```text
  base_retention       = avg_score / 100
  variability_penalty  = min(score_stddev / 50, 0.2)
  retention_rate(0-100) = clamp( (base_retention - variability_penalty) * 100, 0, 100 )
  ```
  
  - مرجع: [impact_metrics_service.py:_calculate_retention](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L1033-L1061)

- تقدم الإتقان (Mastery Progression):
  
  ```text
  weights: EXPERT=4, PROFICIENT=3, APPRENTICE=2, NOVICE=1
  progression_score(0-100) = (EXPERT*4 + PROFICIENT*3 + APPRENTICE*2 + NOVICE*1) / (total * 4) * 100
  ```
  
  - مرجع: [impact_metrics_service.py:_calculate_mastery_progression](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L1064-L1100)

- إنجاز الأهداف (Goal Achievement):
  
  ```text
  proficient_subjects = count(score >= 70 and attempts >= 3)
  total_subjects      = count(subjects with attempts >= 3)
  achievement_rate(%) = (proficient_subjects / total_subjects) * 100
  ```
  
  - مرجع: [impact_metrics_service.py:_calculate_goal_achievement](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L965-L994)

- الفاعلية الشاملة (Effectiveness):
  
  ```text
  overall_effectiveness =
      0.30 * achievement_rate +
      0.25 * velocity_score +
      0.25 * retention_rate +
      0.20 * progression_score
  ```
  
  - مرجع: [impact_metrics_service.py: effectiveness composition](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/services/impact_metrics_service.py#L249-L291)

- المؤشر المركب للأثر العام (للاطلاع):
  
  ```text
  overall_impact = 0.40 * ROI + 0.40 * Effectiveness + 0.20 * (100 - GapScore)
  ```
  
  - مرجع: [analytics/api/impact.py:_calculate_overall_impact](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/api/impact.py#L899-L902)

## Pillar 03 — جودة المحتوى الذكي وامتثاله

**نطاق الركيزة**
- تقييم جودة مخرجات الـ LLM (ملخصات الوحدات، بطاقات تعليمية، أسئلة) مقابل المصدر
- التزام المخرجات بالهيكل المطلوب، اللغة، التغطية، والدقة

**تقييم ملخص الوحدة (Unit Summary)**
- نقطة نهاية التقييم:
  - POST /api/v1/unit-summaries/{summary_id}/evaluate
  - GET /api/v1/unit-summaries/{summary_id}/evaluation
  - مرجع: [unitsummary_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/api/unitsummary_api.py#L311-L368)
- الخدمة المنفذة للتقييم (استخدام LLM كحَكَم وفقًا لروبرك):
  - مرجع: [summary_quality_service.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/summary_quality_service.py)
- الروبرك المعتمدة (p03-summary-v1):
  - Coverage 35% — تغطية الأفكار الرئيسية
  - Factual Accuracy 35% — الدقة مقابل PDF
  - Conciseness 15% — الإيجاز دون إخلال
  - Language Quality 15% — الوضوح وسلامة اللغة والأسلوب
  - مرجع وصفي: [Pillar 03.md](file:///etc/komodo/stacks/vega_2/Pillar%2003.md)
- موجه التقييم (System/User):
  - يُرسل للحكم تعليمات واضحة بإرجاع JSON صارم يحتوي الدرجات والتعليقات والإجمالي الموزون وrubric_version، مع تمرير نص summary وkey_points وexamples، وإرفاق PDF (base64) عند توفره للتحقق من الدقة.

**توليد ملخص الوحدة (للسياق فقط)**
- بناء موجهات RAG (System/User) مع قيود الإخراج الصارمة (JSON جذر، مفاتيح sections بـ UUIDs):
  - مرجع: [llm_prompts.py: build_rag_unit_summary_system_prompt](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/llm_prompts.py#L750-L823)
  - مرجع: [llm_prompts.py: build_rag_unit_summary_user_query](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/llm_prompts.py#L824-L951)
- خدمة التنفيذ مع استخلاص JSON المتين:
  - مرجع: [unitsummary_llm_service.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/unitsummary_llm_service.py)

**تقييم البطاقات التعليمية**
- تقييم فردي/دفعات (Batch) بحسب القسم:
  - مسار جديد للدفعات حسب التعديل الأخير: POST /api/v1/flashcards/sections/{section_id}/evaluate-batch
  - مرجع: [flashcards_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/api/flashcards_api.py)
- الروبرك والأوزان مماثلة في المنهج (Coverage/Accuracy/Conciseness/Language)، ويُخزن الناتج ضمن evaluation_details/quality_score.

**ملاحظات جودة التنفيذ**
- استخلاص JSON متين عبر extract_json_from_content لتجنب أعطال تنسيق ردود LLM.
- ضبط التتبع والاستخدام (AI usage) وربط الاستهلاك بالاعتمادات (Credits) حيثما ينطبق.


## الأذونات والحدود
- معظم مسارات الإدارة/القياس تتطلب أدواراً محددة:
  - Admin، Content Reviewer، Org Admin/Member
- قيود المعدل (Rate Limit) مفعّلة على مسارات التقييم لمنع الإساءة:
  - مثال: 10/minute لتقييمات الملخصات


## أمثلة استهلاك سريعة

**ملخص جودة الأسئلة (ركيزة 01)**
```bash
curl -G 'https://<host>/api/v1/feedback/assessment/summary' \
  --data-urlencode 'artifact_id=<ARTIFACT_UUID>' \
  --data-urlencode 'unit_id=<UNIT_UUID>' \
  --data-urlencode 'max_rating=3' \
  --data-urlencode 'min_ratings_count=2' \
  --data-urlencode 'limit=20' \
  -H 'Authorization: Bearer <TOKEN>'
```

**ROI للمستخدم الحالي (ركيزة 02)**
```bash
curl 'https://<host>/api/v1/analytics/impact/roi/30d' \
  -H 'Authorization: Bearer <TOKEN>'
```

**تقييم ملخص وحدة (ركيزة 03)**
```bash
curl -X POST 'https://<host>/api/v1/unit-summaries/<SUMMARY_ID>/evaluate' \
  -H 'Authorization: Bearer <TOKEN>' \
  -H 'Content-Type: application/json' \
  -d '{"judge":"gemini-2.5-pro","force":true}'
```


## نقاط تشغيلية
- التخزين المؤقت: يستخدم في بعض مسارات التحليلات (ROI/Effectiveness) لمدة 24 ساعة.
- السقوط إلى service_feedback يضمن استمرار الرصد حتى لو تأخر تحديث المؤشرات المجمعة.
- تتبع استهلاك الذكاء الاصطناعي وربطه بالاعتمادات موجود في طبقات التوليد/التقييم.


## مراجع كودية
- Pillar 01 (رضا المستخدم):
  - [assessment/api/feedback_analytics.py](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_analytics.py)
  - [assessment/api/feedback_admin_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/feedback_admin_api.py)
- Pillar 02 (أداء الطلبة):
  - [analytics/api/impact.py](file:///etc/komodo/stacks/vega_2/main-backend/src/analytics/api/impact.py)
  - [assessment/api/irt_evaluation.py](file:///etc/komodo/stacks/vega_2/main-backend/src/assessment/api/irt_evaluation.py)
- Pillar 03 (جودة المحتوى الذكي):
  - التقييم: [content/api/unitsummary_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/api/unitsummary_api.py), [content/api/flashcards_api.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/api/flashcards_api.py)
  - الموجهات: [content/services/llm_prompts.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/llm_prompts.py)
  - التنفيذ والاستخلاص: [content/services/unitsummary_llm_service.py](file:///etc/komodo/stacks/vega_2/main-backend/src/content/services/unitsummary_llm_service.py)
