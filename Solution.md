Задание 1: расчет Rolling retention

with transactions as (
	select 
		u2.user_id,
		date (u2.entry_at) as entry_at,
		date (u.date_joined) as date_joined,
		extract (days from u2.entry_at- u.date_joined) as diff,
		to_char (u.date_joined, 'YYYY-MM') as ym
	from userentry u2
	join users u 
	on u2.user_id=u.id
--период для анализа ограничиваем данными за 2022г, т.к. данные в 2021г очень неоднородные, есть месяцы с сильной «просадкой» в значениях, как будто платформа проходила этап становления/обновления/доработок и стабилизироваться начала только к ноябрю-декабрю 2021 
	where to_char (u.date_joined, 'YYYY')='2022'
	)
select 
	ym,
	round (100.0 * count(distinct case when diff>=0 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day0 (%)",
	round (100.0 * count(distinct case when diff>=1 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day1 (%)",
	round (100.0 * count(distinct case when diff>=3 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day3 (%)",
	round (100.0 * count(distinct case when diff>=7 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day7 (%)",
	round (100.0 * count(distinct case when diff>=14 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day14 (%)",
	round (100.0 * count(distinct case when diff>=30 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day30 (%)",
	round (100.0 * count(distinct case when diff>=60 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day60 (%)",
	round (100.0 * count(distinct case when diff>=90 then user_id end)/ count(distinct case when diff>=0 then user_id end),2) as "day90 (%)"
from transactions
group by ym


Выводы: 
Только около 40% от всех зарегистрировавшихся проявляют активности в самом начале -в 1-ый день после регистрации. И в целом подавляющее большинство активностей наблюдается в течение первого месяца после регистрации, постепенно снижаясь (примерно на 4-5%) в каждом из периодов (n-дней) и в марте-апреле приближаются к 0 уже к концу 2 месяца «жизни» пользователя на платформе. 
Дольше 30 дней, остаются, видимо, те пользователи, которые находятся либо постоянно в процессе саморазвития, либо которые учатся в более медленном темпе.
Нужно анализировать, почему сразу после регистрации такой невысокий процент хотя бы попробовать поработать на платформе (в день 1, день 7). Что им мешает?

Задание 2: расчет метрик относительно баланса пользователя

with total_sum as (
	select 
		t.user_id, 
--с учетом транзакций списания с type_id 1 и 23-28, считаем общие суммы начислений, списаний и баланс
		SUM(case when type_id in (1, 23, 24, 25, 26, 27, 28) then -value end) as wr_off,
		SUM(case when type_id not in (1, 23, 24, 25, 26, 27, 28) then value end) as accrual,
		SUM(case when type_id in (1, 23, 24, 25, 26, 27, 28) then -value else value end) as user_balance
	from "transaction" t 
	group by t.user_id 
)
--рассчитываем средние показатели
select 	
	round (AVG(wr_off), 2) as avg_written_off,
	round (AVG(accrual), 2) as avg_accrual,
	round (AVG(user_balance), 2) as avg_user_balance,
--рассчитываем медиану пользовательского баланса с помощью рекомендованной в задании функции mode within group
	mode() within group (order by user_balance) as median_balance,
--для сравнения, рассчитываем медиану с помощью функции percentile_cont (0.5) within group, где значение 0.5 соответствует медиане
	percentile_cont (0.5) within group (order by user_balance) as median2_balance
from total_sum 

Выводы:
Пользовательский баланс в среднем положительный. Размер среднего начисления существенно выше среднего списания. Выглядит так, что пользователь может «заработать» начислениями больше, чем возникает потребность потратить.  Предположу, что у пользователя нет необходимости тратить больше кодкоинов, т.к. в базовом (бесплатном) функционале и так доступны многие опции. Тут можно в чем-то урезать базовый функционал, перевести часть опций в «платную» категорию.
Значительная разница между средней цифрой баланса и медианной говорит о том, что есть «выбросы» в значениях. Т.е. среди пользователей есть «передовики» обучения, с гораздо большей активностью на платформе (и начислениями), чем у остальных студентов. Возможно, эти «передовики»- группа тестировщиков платформы или обучающиеся от какой-то компании, т.е. люди, которые просто обязаны проявлять повышенную активность в обучении в сравнении со «свободными» студентами.

ЗАДАНИЕ 3: расчет метрик активностей пользователей на платформе
•	Активность по задачам:
--для расчета кол-ва попыток соединяем данные по "выполненным" и "проверенным" (включая не успешные проверки) задачам из двух таблиц
with united_table as (
select 
	user_id,
	problem_id
from coderun c
union all 
select 
	user_id,
	problem_id
from codesubmit
),
--считаем количество успешно решенных пользователями уникальных задач
solved_tasks as (
select 
	user_id,
	count (distinct problem_id) as tasks_solved
from codesubmit c 
where is_false <1
group by c.user_id 
), 
--считаем по каждому пользователю общее количество попыток решения (т.е. все нажатия "выполнить" и "проверить") по всем задачам
attempts_all as(
select 
	u.user_id,
	count(u.problem_id) as att_all
from united_table u
group by u.user_id
),
att_per_task_solv as (
--считаем по каждому пользователю, сколько в среднем приходится попыток на 1 решенную задачу
select 
	a.user_id,
	round (SUM(a.att_all)/SUM(st.tasks_solved):: numeric, 2) as att_per_task
from attempts_all a
join solved_tasks st
on a.user_id =st.user_id 
group by a.user_id
)
--вычисляем средние значения
select 
	round (AVG(st.tasks_solved),2) as avg_tasks_solv,
	round (AVG(apts.att_per_task),2) as avg_att_per_task
from solved_tasks st, att_per_task_solv apts

Выводы: 
Т.к. на решение задач у пользователей тратится больше усилий (что видно по кол-ву попыток на 1 задачу- 14,52), то стоит сфокусировать внимание на пересмотр доступа к бесплатным подсказкам и решениям в первую очередь для задач. И в целом видно, что работа с задачами для пользователей- гораздо более ценный опыт, чем с тестами.
	
•	Активность по тестам:
with uniq_tests_cnt as (
--считаем, сколько каждый пользователь проходит уникальных тестов
select 
	user_id,
	count(distinct test_id) as uniq_cnt_tests
from teststart t 
group by user_id
),
total_tests_cnt as (
--считаем, сколько всего каждый пользователь делал попыток пройти тесты
select 
	user_id,
	count (test_id) as total_cnt_tests
from teststart t 
group by user_id
),
attempts_per_test as (
--считаем по каждому пользователю, сколько в среднем делалось попыток на 1 уникальный тест
select
	tc.user_id,
	SUM(tc.total_cnt_tests)/SUM(uc.uniq_cnt_tests) :: numeric as att_per_test
from total_tests_cnt tc
join uniq_tests_cnt uc
on tc.user_id=uc.user_id 
group by tc.user_id
)
--считаем средние значения
select 
	round (AVG(uc.uniq_cnt_tests),2) as avg_uniq_tests,
	round (AVG(apt.att_per_test),2) as avg_att_per_test
from uniq_tests_cnt uc, attempts_per_test apt

Выводы: 
О показателях для тестов. Если на платформе доступны 1-2 бесплатных теста, то получившиеся средние цифры прохождений (1,68)/попыток (1,15) вполне объяснимы. Если количество бесплатных тестов больше, тогда возникает вопрос, почему пользователям не интересно их проходить. 


•	Доля от общего числа пользователей, решавшая хотя бы одну задачу или начавшая проходить хотя бы один тест
--для вывода списка пользователей, которые в принципе пробовали решать задачи или пройти тесты, соединяем данные двух таблиц
with united_table as(
select 
user_id,
problem_id
from coderun c
union all 
select 
user_id,
problem_id
from codesubmit
),
--считаем количество уникальных пользователей, присутствующих в этой таблице
users_worked as (
select 
count (distinct u.user_id) as cnt_users_worked
from united_table u
),
--считаем общее количество пользователей, зарегистрированных в базе
users_total as (
select
	count (u.id) as cnt_users_total
from users u 
)
--рассчитываем долю работавших с задачами или тестами от общего количества зарегистрированных пользователей
select 
	round ((uw.cnt_users_worked/ut.cnt_users_total :: numeric)*100 ,2) as worked_users_share
from users_worked uw, users_total ut

Выводы: 
С одной стороны, если учитывать в анализе весь период данных, доступный в базе, доля довольно низкая 33,38%. И надо выяснять, что мешает 66% зарегистрировавшимся пользователям, которые потратили время на регистрацию, хотя бы попробовать решить задачу/начать тест. 
С другой стороны, если не брать в расчет период некой стабилизации платформы и очень неравномерных активностей в этот момент в 2021г, а посмотреть на более стабильный период в 1 квартале 2022г, то картина будет гораздо лучше-63,68% (код дополнительного запроса ниже). При этом все равно стоит проанализировать, что все-таки мешает порядка 33% зарегистрированных сделать хотя бы 1 попытку решить задачу/начать тест. 
Код дополнительного запроса:
--выводим список id пользователей, зарегистрировавшихся в 2022г
with joined_in_2022 as (
select 
	id as joined_in_2022
from users u 
where to_char(u.date_joined , 'YYYY')= '2022' 
),
--для подсчета кол-ва пользователей, попробовавших решить хотя бы 1 задачу/пройти 1 тест объединяем таблицы по задачам и тестам
united_table as (
select
	c.user_id,
	c.created_at
from coderun c 
union all 
select 
	c2.user_id,
	c2.created_at
from codesubmit c2 
union all 
select 
	t.user_id,
	t.created_at
from teststart t 
),
--считаем кол-во активностей уникальных пользователей из списка зарегистрировавшихся в 2022 году, которые присутствуют в объединенной таблице 
worked_in_2022 as (
select 
	count(distinct user_id) as cnt_worked_in_2022
from united_table u
left join joined_in_2022
on u.user_id=joined_in_2022
where joined_in_2022 is not null
),
--считаем кол-во зарегистрировавшихся в 2022
cnt_joined_in_2022 as (
select 
	count (joined_in_2022) as cnt_joined_in_2022
from joined_in_2022
)
--рассчитываем долю хотя бы что-то сделавших на платформе из зарегистрированных в 2022г
select 
	round ((w.cnt_worked_in_2022/ cj.cnt_joined_in_2022 :: numeric ) * 100, 2) as worked_share_v2
from worked_in_2022 w, cnt_joined_in_2022 cj

•	Информация по покупкам материалов за кодкоины
with a as (
--считаем кол-во человек (уникальных пользователей), открывавших задачи за кодкоины, т.е. с типом транзации=23(Купить задачу)
select 
	count (distinct user_id) cnt_users_open_tasks
from "transaction" t 
where t.type_id =23
),
b as (
--считаем кол-во человек, открывавших тесты за кодкоины, т.е. с типом транзации=26 и 27 (Купить тест и Купить юнит-тест)
select 
	count (distinct user_id) cnt_users_open_tests
from "transaction" t 
where t.type_id =26 or t.type_id=27
),
c as (
--считаем кол-во человек, открывавших подсказки за кодкоины, т.е. с типом транзации=24 (Купить подсказку к задаче)
select 
	count (distinct user_id) cnt_users_open_tips
from "transaction" t 
where t.type_id =24
),
d as (
--считаем кол-во человек, открывавших решения за кодкоины, т.е. с типом транзации=25 и 28 (Купить решение задачи и Купить ответ на вопрос теста)
select 
	count (distinct user_id) cnt_users_open_solutions
from "transaction" t 
where t.type_id =25 or t.type_id=28
),
e as (
select 
--считаем кол-во подсказок/тестов/задач/решений, открытых за кодкоины разными пользователями
	count (*) cnt_all_for_coins,
--считаем кол-во пользователей, которые хотя бы что-нибудь покупали за кодкоины 
	count (distinct t.user_id ) users_bying_for_coins
from "transaction" t 
where t.type_id between 23 and 28
),
f as (
--считаем кол-во пользователей, которые имеют хотя бы 1 транзакцию
select 
	count (distinct t.user_id ) users_bying_any_way
from "transaction" t 
)
select 
	a.cnt_users_open_tasks,
	b.cnt_users_open_tests,
	c.cnt_users_open_tips,
	d.cnt_users_open_solutions,
	e.cnt_all_for_coins,
	e.users_bying_for_coins,
	f.users_bying_any_way
from a, b, c, d, e, f 

Выводы:
В базе всего зарегистрировано 2774 пользователя. 
Получается, что порядка 87% зарегистрированных совершили хотя бы 1 транзакцию. Примерно половина из них- за кодкоины.
В основном кодкоины используют на открытие платных задач и тестов. При этом по количеству пользователей, лидируют покупающие тесты (676 человек), за ними идут покупающие задачи (522 человека).
Но если смотреть на разбивку в разрезе количества купленных задач и тестов (код дополнительного запроса ниже), то чаще покупают задачи (1675 шт), чем тесты (989 шт).
Решения за кодкоины не очень востребованы, подсказки совсем мало. Возможно потому, что много подсказок/решений доступно в бесплатном режиме (либо, что маловероятно, на мой взгляд, многие пользователи очень упорные и решают задачи через множество попыток, не прибегая к подсказкам).
Код дополнительного запроса: 
--считаем, cколько в общем подсказок/тестов/задач/решений было открыто за кодкоины
with all_for_coins as (
select 
	count (*) cnt_all_for_coins
from "transaction" t 
where t.type_id between 23 and 28
),
--cколько подсказок было открыто за кодкоины
tips as (
select 
	count (*) cnt_tips_for_coins
from "transaction" t 
where t.type_id= 24
),
--cколько тестов было открыто за кодкоины
tests as (
select 
	count (*) cnt_tests_for_coins
from "transaction" t 
where t.type_id between 26 and 27
),
--cколько задач было открыто за кодкоины
prb as (
select 
	count (*) cnt_prb_for_coins
from "transaction" t 
where t.type_id= 23
),
--cколько решений было открыто за кодкоины
solutions as (
select 
	count (*) cnt_solut_for_coins
from "transaction" t 
where t.type_id= 25 or t.type_id= 28
)
--собираем значения в одну таблицу
select 
	a.cnt_all_for_coins,
	t.cnt_tips_for_coins,
	tst.cnt_tests_for_coins,
	p.cnt_prb_for_coins,
	s.cnt_solut_for_coins
from all_for_coins a, tips t, tests tst, prb p, solutions s
