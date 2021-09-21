Задача:
Понять, какие каналы привлечения эффективны и неэффективны в метрике LTV, и отследить динамику этой метрики.

Конкретные шаги:
Шаг 1: Выбрать модель расчёта LTV.
Шаг 2: Оценить затраты по date/source/medium/campaign.
Шаг 3: Оценить доходы по date/source/medium/campaign.
Шаг 4: Оценить LTV по date/source/medium/campaign.
Шаг 5: Построить отчёт в PBI с отображением LTV в разбивке по source/medium/campaign и дате привлечения пользователя для отслеживания динамики изменений.

WITH  mrkt_costs_corrected AS
    (
    SELECT
      mc.date cohort_date,
      LOWER (mc.source) source,
      mc.medium,
      mc.campaign,
      mc.costs_rub
    FROM gd1.marketing_costs mc),
	
currency_rates_correct_date AS
    (
    SELECT
      cr.date::date correct_date,
      cr.usd_to_rub
    FROM gd1.currency_rates cr),
transactions_rub AS
	(SELECT
     	tr.id,
     	tr.created_at,
     	tr.uuid,
     	(tr.subtotal_cents::numeric * crcd.usd_to_rub::numeric)/100 subtotal_rub,
     	(tr.total_cents::numeric * crcd.usd_to_rub::numeric)/100 total_rub
    FROM gd1.transactions tr
    JOIN currency_rates_correct_date crcd ON tr.created_at::date=crcd.correct_date::date),
	
transactions_with_acq_info AS
	(SELECT
		tru.id,
		tru.created_at transactions_date,
		tru.subtotal_rub,
		tru.subtotal_rub-tru.total_rub discount,
		sa.start_at first_transaction_date,
		sa.source,
		sa.medium,
		sa.campaign
	FROM transactions_rub tru
	JOIN gd1.sessions_acquisition sa ON tru.uuid=sa.uuid
	WHERE (tru.created_at-sa.start_at)::integer<=30),
	
data_per_cohort AS
	(SELECT
		twai.first_transaction_date cohort_date,
		twai.source,
	 	twai.medium,
	 	twai.campaign,
	 	SUM(twai.subtotal_rub) income,
	 	SUM(twai.discount) discounts
	 FROM transactions_with_acq_info twai
	 JOIN mrkt_costs_corrected mcc
	 ON twai.first_transaction_date=mcc.cohort_date
	 AND twai.source=mcc.source
	 AND twai.medium=mcc.medium
	 AND twai.campaign=mcc.campaign
	 GROUP BY 1,2,3,4
	 ORDER BY 1),
	
 ltv_30_days AS
 	(SELECT
		mcc.cohort_date,
	 	mcc.source,
	 	mcc.medium,
	 	mcc.campaign,
	 	dpc.income-mcc.costs_rub-dpc.discounts LTV
	 FROM
		mrkt_costs_corrected mcc
	 LEFT JOIN data_per_cohort dpc
	 ON mcc.cohort_date=dpc.cohort_date
	 AND mcc.source=dpc.source
	 AND mcc.medium=dpc.medium
	 AND mcc.campaign=dpc.campaign)
	
SELECT
    ld.cohort_date,
    ld.source,
	  ld.medium,
	  ld.campaign,
	  ROUND(ld.LTV)	as LTV
FROM ltv_30_days ld
    
  https://app.powerbi.com/reportEmbed?reportId=40572edf-49e6-4e33-9cf4-7e71dd958783&autoAuth=true&ctid=6a4dee01-c3f5-4d4b-bdd2-9e1f1482ac5d&config=eyJjbHVzdGVyVXJsIjoiaHR0cHM6Ly93YWJpLXdlc3QtZXVyb3BlLWQtcHJpbWFyeS1yZWRpcmVjdC5hbmFseXNpcy53aW5kb3dzLm5ldC8ifQ%3D%3D
