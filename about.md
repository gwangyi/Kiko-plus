---
title: About
permalink: /about/
---

## Sungkwang Lee

#### Employment

{% for job in site.data.resume.jobs %}
{{ job.title }}
: _{{ job.company }}_ &#x2015; {{ job.period }}
{% for project in job.projects %}

  {{ project }}
{% endfor %}
{% endfor %}

#### Education

{% for edu in site.data.resume.education %}
{{ edu.qualification }}
: _{{ edu.school }}_ &#x2015; {{ edu.period }}
{% for item in edu.items %}
  {{ item }}
{% endfor %}

{% endfor %}

#### Miscellaneous Experience

{% for exp in site.data.resume.experience %}
{{ exp.title }}
: {{ exp.detail }}

{% endfor %}

#### Professional Qualifications

{% for qual in site.data.resume.qualifications %}
{{ qual.title }}
: {{ qual.detail }}

{% endfor %}

#### Foreign Languages

{% for lang in site.data.resume.languages %}
{{ lang.language }}
: {{ lang.grade }}

{% endfor %}

#### Programming Skills

{% for prog in site.data.resume.programming %}
{{ prog.grade }}
: {{ prog.title }}

{% endfor %}

#### Interests

{% for interest in site.data.resume.interests %}
- {{ interest }}
{% endfor %}

#### Publications

##### Conference

{% for paper in site.data.resume.conference_paper %}
* {{ paper.author }}, “**{{ paper.title }}**,” _{{ paper.pub }}_, {{ paper.date }}. [doi:{{ paper.doi }}](http://dx.doi.org/{{ paper.doi }})
{% endfor %}

##### Journal

{% for paper in site.data.resume.journal_paper %}
* {{ paper.author }}, “**{{ paper.title }}**,” _{{ paper.pub }}_, {{ paper.date }}. [doi:{{ paper.doi }}](http://dx.doi.org/{{ paper.doi }})
{% endfor %}

#### Awards

{% for award in site.data.resume.awards %}
* **{{ award.title }}**  
  {{ award.detail }}
{% endfor %}

