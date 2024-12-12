<b>SCENARIO 1 - Min, Max, and Average Closing Price</b>


SELECT
    security,
    MIN(close_price)||' $' AS min_price,
    MAX(close_price)||' $' AS max_price,
    ROUND(AVG(close_price),2)||'%' AS avg_price
FROM 
    price_data
GROUP BY 
    security;


<b>SCENARIO 1 - Most Significant Positive Spike</b>


WITH price_spikes AS (
    SELECT 
        security, 
        date, 
        close_price,
        close_price - LAG(close_price) OVER (PARTITION BY security ORDER BY date) AS price_spike
    FROM 
        price_data
),
ranked_spikes AS (
    SELECT 
        security, 
        date, 
        close_price, 
        price_spike,
        ROW_NUMBER() OVER (PARTITION BY security ORDER BY price_spike DESC) AS rank
    FROM 
        price_spikes
    WHERE 
        price_spike IS NOT NULL
)
SELECT 
    security, 
    date, 
    close_price, 
    price_spike
FROM 
    ranked_spikes
WHERE 
    rank = 1;


<b>SCENARIO 1 - ROI using the most significant spike's date</b>


WITH base_data AS (
    SELECT 
		security,
        MIN(close_price) AS buy_price,
        MAX(close_price) AS spike_price
    FROM 
        price_data
    WHERE 
        date <= CURRENT_DATE
    GROUP BY 
        security
)
SELECT 
    security,
    1000 * (spike_price - buy_price) AS roi
FROM 
    base_data;


----------------------------------------------


<b>SCENARIO 2 - Trade Quantity vs. Holdings</b>


SELECT 
    h.security_id,
    h.end_quantity AS holdings_quantity,
    COALESCE(SUM(CAST(t.trade_quantity AS INTEGER)), 0) AS total_trade_quantity,
    h.end_quantity - COALESCE(SUM(CAST(t.trade_quantity AS INTEGER)), 0) AS discrepancy
FROM 
    holdings h
LEFT JOIN 
    trades t 
ON 
    h.security_id = t.security_id
GROUP BY 
    h.security_id, h.end_quantity
HAVING 
    h.end_quantity != COALESCE(SUM(CAST(t.trade_quantity AS INTEGER)), 0);


<b>SCENARIO 2 - Validate Net Proceeds</b>


SELECT 
    unit_price,
    net_proceeds AS recorded_proceeds,
    trade_quantity * unit_price AS calculated_proceeds,
    net_proceeds - (trade_quantity * unit_price) AS discrepancy
FROM 
    trades
WHERE 
    net_proceeds != unit_price * trade_price;
