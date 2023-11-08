---
title: "ADAE"
author: "Stanislav Legkovoy"
output: 
  html_document: 
    keep_md: yes
date: "2023-11-08"
---




```r
adsl <- read.xlsx("ADSL.xlsx")
preae <- read.xlsx("sdtm-like files/AE_ADVERSEEVENTS.xlsx")
termtran <- read.xlsx("sdtm-like files/terms_translation.xlsx")
ae <- preae %>% 
  left_join(termtran, by=join_by("AEBODSYS"=="SOC", "AEDECOD"=="PT")) %>% 
  mutate(SOCT = paste0("(", SOCT, ")")) %>% 
  mutate(AEBODSYS = paste0(AEBODSYS, SOCT))
```


```r
adae <- ae %>%
  select(-STUDYID) %>% 
  left_join(adsl, by=join_by("SUBJID")) %>% 
  mutate(across(ends_with("DT"), ~gsub('\\.', '-', .))) %>% 
  mutate(APERIOD = as.integer(if_else(as.Date(AESTDTC) >= as.Date(AP01SDT, format = "%d-%m-%Y") & 
                           as.Date(AESTDTC) <= as.Date(AP01EDT, format = "%d-%m-%Y"), 1, if_else(as.Date(AESTDTC) >= as.Date(AP02SDT, format = "%d-%m-%Y") & as.Date(AESTDTC) <= as.Date(AP02EDT, format = "%d-%m-%Y"), 2, NA_integer_))),
         APERIODC = if_else(APERIOD == 1, 'Период 1', if_else(APERIOD == 2, 'Период 2', NA_character_)),
         AP01SDT = format(as.Date(AP01SDT, format = "%d-%m-%Y"), "%d.%m.%Y"),
         AP01EDT = format(as.Date(AP01EDT, format = "%d-%m-%Y"), "%d.%m.%Y"),
         AP02SDT = format(as.Date(AP02SDT, format = "%d-%m-%Y"), "%d.%m.%Y"),
         AP02EDT = format(as.Date(AP02EDT, format = "%d-%m-%Y"), "%d.%m.%Y"),
         ASTDT = format(as.Date(AESTDTC), "%d.%m.%Y"),
         AENDT = format(as.Date(AEENDTC), "%d.%m.%Y"),
         ADURN = as.integer(as.Date(AENDT, format="%d.%m.%Y") - as.Date(ASTDT, format="%d.%m.%Y") + 1),
         ADURU = "день",
         AESER = if_else(AESER == 'Да', 'Y', if_else(AESER=='Нет', 'N', NA_character_)),
         ASEV = AESEV, 
         TRTSEQPN = as.integer(TRTSEQPN),
         ASEVN = as.integer(case_when(AESEV == 'Легкое' ~ 1,
                           AESEV == 'Среднее' ~ 2,
                           AESEV == 'Тяжёлое' ~ 3)),
         TRTEMFL = if_else((as.Date(ASTDT, format="%d.%m.%Y") >= as.Date(AP01SDT, format = "%d-%m-%Y") & as.Date(ASTDT, format="%d.%m.%Y") <= as.Date(AP01EDT, format = "%d-%m-%Y")) | (as.Date(ASTDT, format="%d.%m.%Y") >= as.Date(AP02SDT, format = "%d-%m-%Y") & as.Date(ASTDT, format="%d.%m.%Y") <= as.Date(AP02EDT, format = "%d-%m-%Y")), "Y", NA_character_),
         PREFL = if_else(as.Date(ASTDT, format="%d.%m.%Y") < as.Date(AP01SDT, format = "%d-%m-%Y"), "Y", NA_character_),
         TRTP = if_else(APERIOD == 1, TRT01P, if_else(APERIOD == 2, TRT02P, NA_character_)),
         TRTPN = as.integer(if_else(APERIOD == 1, TRT01PN, if_else(APERIOD == 2, TRT02PN, NA_integer_))),
         AERELN = as.integer(case_when(AEREL == 'Определенная' ~ 1,
                            AEREL == 'Вероятная' ~ 2,
                            AEREL == 'Возможная' ~ 3,
                            AEREL == 'Сомнительная' ~ 4,
                            AEREL == 'Условная' ~ 5,
                            AEREL == 'Не классифицируемая' ~ 6,
                            AEREL == 'Не связано' ~7)),
         RELGR1 = case_when(AEREL %in% c('Определенная', 'Вероятная', 'Возможная', 'Сомнительная', 'Условная') ~ 'Связано',
                            AEREL %in% c(NA, "Не классифицируемая") ~ NA_character_,
                            AEREL == 'Не связано' ~ 'Не связано'),
         RELGR1N = as.integer(if_else(RELGR1 == 'Не связано', 0, if_else(RELGR1 == 'Связано', 1, 2))),
         AERES = AEOUT,
         AERESN = as.integer(case_when(AERES == 'Выздоровление без осложнений' ~ 1,
                            AERES == 'Стадия выздоровления' ~ 2,
                            AERES == 'Без изменений' ~ 3,
                            AERES == 'Выздоровление с осложнениями' ~ 4,
                            AERES == 'Смерть' ~ 5,
                            AERES == 'Не известно' ~ 6)), 
         AECMFL = if_else(AECONTRT == 'Да', 'Y', 'N'),
         APHASE = case_when(PREFL == "Y" ~ "Скрининг", 
                            TRTEMFL == "Y" ~ "Лечение",
                            .default = NA_character_),
         AEENRF = if_else(AEENRTPT == 'ONGOING', 'ONGOING', NA_character_),
         Y_ST = year(as.Date(ASTDT, format="%d.%m.%Y")),
         M_ST = month(as.Date(ASTDT, format="%d.%m.%Y")),
         D_ST = day(as.Date(ASTDT, format="%d.%m.%Y")),
         Y_EN = year(as.Date(AENDT, format="%d.%m.%Y")),
         M_EN = month(as.Date(AENDT, format="%d.%m.%Y")),
         D_EN = day(as.Date(AENDT, format="%d.%m.%Y")),
         WEIGHT = WEIGHTBL,
         AGE = as.integer(AGE),
         ASTDTF = case_when(is.na(ASTDT) | is.na(Y_ST) ~ "Y",
                            is.na(M_ST) ~ "M",
                            is.na(D_ST) ~ "D"),
         AENDTF = case_when(is.na(AENDT) | is.na(Y_EN) ~ "Y",
                            is.na(M_EN) ~ "M",
                            is.na(D_EN) ~ "D"),
         )%>%
  select(STUDYID, SUBJID, USUBJID, SITEID, TRTSEQP, TRTSEQPN, AP01SDT, AP01EDT,
           AP02SDT, AP02EDT, APERIOD, APERIODC, TRTEMFL, PREFL, TRTP, TRTPN,
           AESEQ, AETERM, AEBODSYS, AEDECOD, AESTDTC, ASTDT, ASTDTF, AEENDTC,
           AENDT, AENDTF, AEENRTPT, AEENRF, ADURN, ADURU, AESER, APHASE, ASEV,
           ASEVN, AEREL, AERELN, RELGR1, RELGR1N, AEACN, AERES, AERESN, AECMFL,
           SAFFL, AGE, SEX, WEIGHT, RACE) %>%
  relocate(STUDYID, SUBJID, USUBJID, SITEID, TRTSEQP, TRTSEQPN, AP01SDT, AP01EDT,
           AP02SDT, AP02EDT, APERIOD, APERIODC, TRTEMFL, PREFL, TRTP, TRTPN,
           AESEQ, AETERM, AEBODSYS, AEDECOD, AESTDTC, ASTDT, ASTDTF, AEENDTC,
           AENDT, AENDTF, AEENRTPT, AEENRF, ADURN, ADURU, AESER, APHASE, ASEV,
           ASEVN, AEREL, AERELN, RELGR1, RELGR1N, AEACN, AERES, AERESN, AECMFL,
           SAFFL, AGE, SEX, WEIGHT, RACE)

str(adae)
```

```
## 'data.frame':	11 obs. of  47 variables:
##  $ STUDYID : chr  "XXXXX-XX-XX" "XXXXX-XX-XX" "XXXXX-XX-XX" "XXXXX-XX-XX" ...
##  $ SUBJID  : chr  "09005" "09005" "09006" "09006" ...
##  $ USUBJID : chr  "XXXXX-XX-XX-09005" "XXXXX-XX-XX-09005" "XXXXX-XX-XX-09006" "XXXXX-XX-XX-09006" ...
##  $ SITEID  : chr  "09" "09" "09" "09" ...
##  $ TRTSEQP : chr  "RT" "RT" "RT" "RT" ...
##  $ TRTSEQPN: int  2 2 2 2 2 1 1 1 2 2 ...
##  $ AP01SDT : chr  "26.06.2023" "26.06.2023" "27.06.2023" "27.06.2023" ...
##  $ AP01EDT : chr  "29.06.2023" "29.06.2023" "30.06.2023" "30.06.2023" ...
##  $ AP02SDT : chr  "03.07.2023" "03.07.2023" "04.07.2023" "04.07.2023" ...
##  $ AP02EDT : chr  "06.07.2023" "06.07.2023" "07.07.2023" "07.07.2023" ...
##  $ APERIOD : int  1 1 1 1 1 2 2 2 NA 1 ...
##  $ APERIODC: chr  "Период 1" "Период 1" "Период 1" "Период 1" ...
##  $ TRTEMFL : chr  NA NA NA NA ...
##  $ PREFL   : chr  NA NA NA NA ...
##  $ TRTP    : chr  "drug2" "drug2" "drug2" "drug2" ...
##  $ TRTPN   : int  2 2 2 2 2 2 2 2 NA 2 ...
##  $ AESEQ   : chr  "1" "2" "1" "1" ...
##  $ AETERM  : chr  "TRANSITORY BURNING IN SITE OF APPLICATION" "APPLICATION SITE BURNING" "BURNING SENSATION WHEN URINATING" "URINATION PAIN" ...
##  $ AEBODSYS: chr  "General disorders and administration site conditions(Общие нарушения и реакции в месте введения)" "General disorders and administration site conditions(Общие нарушения и реакции в месте введения)" "Renal and urinary disorders(Нарушения со стороны почек и мочевыводящих путей)" "Renal and urinary disorders(Нарушения со стороны почек и мочевыводящих путей)" ...
##  $ AEDECOD : chr  "Administration site irritation " "Administration site irritation " "Dysuria" "Dysuria" ...
##  $ AESTDTC : chr  "2023-06-28" "2023-06-29" "2023-06-29" "2023-06-29" ...
##  $ ASTDT   : chr  "28.06.2023" "29.06.2023" "29.06.2023" "29.06.2023" ...
##  $ ASTDTF  : chr  NA NA NA NA ...
##  $ AEENDTC : chr  "2023-06-28" "2023-06-29" "2023-06-29" "2023-06-29" ...
##  $ AENDT   : chr  "28.06.2023" "29.06.2023" "29.06.2023" "29.06.2023" ...
##  $ AENDTF  : chr  NA NA NA NA ...
##  $ AEENRTPT: chr  "BEFORE" "BEFORE" "BEFORE" "COINCIDENT" ...
##  $ AEENRF  : chr  NA NA NA NA ...
##  $ ADURN   : int  1 1 1 1 1 1 1 1 1 1 ...
##  $ ADURU   : chr  "день" "день" "день" "день" ...
##  $ AESER   : chr  "N" "N" "N" "N" ...
##  $ APHASE  : chr  NA NA NA NA ...
##  $ ASEV    : chr  "Легкое" "Легкое" "Легкое" "Легкое" ...
##  $ ASEVN   : int  1 1 1 1 1 1 1 1 1 1 ...
##  $ AEREL   : chr  "Определенная" "Вероятная" "Сомнительная" "Возможная" ...
##  $ AERELN  : int  1 2 4 3 7 7 7 7 3 3 ...
##  $ RELGR1  : chr  "Связано" "Связано" "Связано" "Связано" ...
##  $ RELGR1N : int  1 1 1 1 0 0 0 0 1 1 ...
##  $ AEACN   : chr  "Без изменений" "Без изменений" "Без изменений" "Без изменений" ...
##  $ AERES   : chr  "Выздоровление без последствий" "Выздоровление без последствий" "Выздоровление без последствий" "Выздоровление без последствий" ...
##  $ AERESN  : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ AECMFL  : chr  "N" "N" "N" "N" ...
##  $ SAFFL   : chr  "Y" "Y" "Y" "Y" ...
##  $ AGE     : int  38 38 33 33 33 28 28 28 23 23 ...
##  $ SEX     : chr  "Мужской" "Мужской" "Мужской" "Мужской" ...
##  $ WEIGHT  : num  89.2 89.2 82.1 82.1 82.1 81.4 81.4 81.4 72.2 72.2 ...
##  $ RACE    : chr  "Европеоидная" "Европеоидная" "Европеоидная" "Европеоидная" ...
```

```r
write.xlsx(x = adae, file = "ADAE.xlsx")
```

Что увидел:

ASEV: в спецификации и в данных категории тяжести нежелательных явлений не соответсвуют друг другу по форме (женский и средний рода соответственно);

AETERM: в AE есть AETERM, не TERM переменная

ADURN: для вычисления длительности нежелательного явления необходимо из даты конца вычитать дату начала присовокупляя 1 в конце;

AERES/AERESN: в данных не представлена категория среди упомянутых в спецификации;

AREPIOD: надо APERIOD (в спеке опечатка);

P.S.: вероятно перебдел, но на всякий привёл типы переменных в соответствии со спекой, разделяя integer 
и float переменные. в SAS такой промблемы нет, конечно, там только character and numeric...лейблы на переменные вешать уже не стал.

