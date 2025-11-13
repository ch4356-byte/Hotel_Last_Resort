## Summary
This document outlines 8 strategic management queries base on the ERD we made.

## Proposed Management Queries

### 1. Revenue Analysis by Time Period
Purpose: Track revenue trends and seasonal patterns
Business Value: Enables pricing strategy adjustments and resource planning

```sql
-- Monthly revenue breakdown with year-over-year comparison
SELECT
    DATE_FORMAT(b.billing_date, '%Y-%m') as month_year,
    SUM(b.total_amount - b.discount_amount) as net_revenue,
    COUNT(DISTINCT b.billingId) as total_bills,
    AVG(b.total_amount - b.discount_amount) as avg_bill_amount,
    SUM(CASE WHEN r.check_out_date IS NOT NULL THEN 1 ELSE 0 END) as completed_stays
FROM Billing b
LEFT JOIN Reservation r ON b.reservationId = r.resevationId
WHERE b.billing_date >= DATE_SUB(CURDATE(), INTERVAL 15 MONTH)
GROUP BY DATE_FORMAT(b.billing_date, '%Y-%m')
ORDER BY month_year DESC;
```

### 2. Room Occupancy Rate Analysis
Purpose: Monitor facility utilization and identify optimization opportunities
Business Value: Maximizes revenue per available room and identifies underutilized assets

```sql
-- Daily occupancy rates by wing and room category
SELECT
    w.code as wing_code,
    b.name as building_name,
    rm.room_category,
    COUNT(DISTINCT rm.roomId) as total_rooms,
    COUNT(DISTINCT CASE WHEN r.status = 'confirmed'
          AND CURDATE() BETWEEN r.check_in_date AND r.check_out_date
          THEN r.roomId END) as occupied_rooms,
    ROUND(COUNT(DISTINCT CASE WHEN r.status = 'confirmed'
          AND CURDATE() BETWEEN r.check_in_date AND r.check_out_date
          THEN r.roomId END) * 100.0 / COUNT(DISTINCT rm.roomId), 2) as occupancy_rate
FROM room rm
JOIN wing w ON rm.wingId = w.wingId
JOIN building b ON w.buildingId = b.buildingId
LEFT JOIN Reservation r ON rm.roomId = r.roomId
GROUP BY w.code, b.name, rm.room_category
ORDER BY occupancy_rate DESC;
```

### 3. Top Revenue Generating Customers
Purpose: Identify high-value customers for targeted retention programs
Business Value: Supports customer relationship management and loyalty program development

```sql
-- Top 20 customers by lifetime revenue with recent activity
SELECT
    c.first_name,
    c.last_name,
    c.organization,
    COUNT(DISTINCT b.billingId) as total_bookings,
    SUM(b.total_amount - b.discount_amount) as lifetime_revenue,
    MAX(b.billing_date) as last_visit_date,
    AVG(b.total_amount - b.discount_amount) as avg_spend_per_visit,
    COUNT(DISTINCT CASE WHEN b.billing_date >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
          THEN b.billingId END) as bookings_last_12_months
FROM ContactInfo c
JOIN Billing b ON c.contactId = b.contactId
GROUP BY c.contactId, c.first_name, c.last_name, c.organization
HAVING lifetime_revenue > 0
ORDER BY lifetime_revenue DESC
LIMIT 20;
```

### 4. Service Revenue Breakdown
Purpose: Analyze ancillary revenue streams and service popularity
Business Value: Identifies profitable services and opportunities for service expansion

```sql
-- Revenue by service type with usage frequency
SELECT
    s.service_type,
    COUNT(*) as total_transactions,
    SUM(s.amount) as total_revenue,
    AVG(s.amount) as avg_transaction_value,
    COUNT(DISTINCT s.guestId) as unique_customers,
    ROUND(SUM(s.amount) / COUNT(DISTINCT s.guestId), 2) as revenue_per_customer
FROM Service s
WHERE s.date >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
GROUP BY s.service_type
ORDER BY total_revenue DESC;
```

### 5. Event Performance Analytics
Purpose: Evaluate event hosting success and facility utilization
Business Value: Optimizes event pricing and identifies successful event patterns

```sql
-- Event performance with associated guest revenue
SELECT
    e.name as event_name,
    e.start_time,
    e.end_time,
    e.attendance,
    e.guests as estimated_guests,
    COUNT(DISTINCT er.roomId) as rooms_used,
    SUM(er.rate * COALESCE(er.discount, 1)) as event_room_revenue,
    COUNT(DISTINCT r.resevationId) as associated_reservations,
    SUM(b.total_amount - b.discount_amount) as total_associated_revenue
FROM Event e
LEFT JOIN EventRoom er ON e.eventId = er.eventId
LEFT JOIN Reservation r ON e.reservationId = r.resevationId
LEFT JOIN Billing b ON r.resevationId = b.reservationId
WHERE e.start_time >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
GROUP BY e.eventId, e.name, e.start_time, e.end_time, e.attendance, e.guests
ORDER BY total_associated_revenue DESC;
```

### 6. Room Status and Maintenance Analysis
Purpose: Monitor operational efficiency and maintenance needs
Business Value: Reduces downtime and improves guest satisfaction

```sql
-- Current room status distribution with maintenance history
SELECT
    rsh.status,
    COUNT(*) as room_count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM room), 2) as percentage_of_total,
    AVG(TIMESTAMPDIFF(DAY, rsh.start_ts, COALESCE(rsh.end_ts, NOW()))) as avg_days_in_status
FROM room_status_history rsh
WHERE rsh.end_ts IS NULL OR rsh.end_ts = (
    SELECT MAX(rsh2.end_ts)
    FROM room_status_history rsh2
    WHERE rsh2.roomId = rsh.roomId
)
GROUP BY rsh.status
ORDER BY room_count DESC;
```

### 7. Reservation Patterns and Booking Lead Time
Purpose: Understand booking behavior for demand forecasting
Business Value: Improves revenue management and capacity planning

```sql
-- Booking patterns with lead time analysis
SELECT
    CASE
        WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 7 THEN '0-7 days'
        WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 30 THEN '8-30 days'
        WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 90 THEN '31-90 days'
        ELSE '90+ days'
    END as booking_lead_time,
    COUNT(*) as reservation_count,
    AVG(b.total_amount - b.discount_amount) as avg_revenue,
    AVG(r.party_number) as avg_party_size,
    ROUND(AVG(TIMESTAMPDIFF(DAY, r.check_in_date, r.check_out_date)), 1) as avg_stay_length
FROM Reservation r
JOIN Billing b ON r.resevationId = b.reservationId
WHERE r.reservation_date >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY CASE
    WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 7 THEN '0-7 days'
    WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 30 THEN '8-30 days'
    WHEN TIMESTAMPDIFF(DAY, r.reservation_date, r.check_in_date) <= 90 THEN '31-90 days'
    ELSE '90+ days'
END
ORDER BY reservation_count DESC;
```

### 8. Facility Utilization and Revenue per Square Foot
Purpose: Evaluate space efficiency and identify underperforming areas
Business Value: Optimizes facility layout and identifies expansion opportunities

```sql
-- Facility utilization with associated revenue
SELECT
    f.facilitiy_name,
    f.facility_type,
    f.capacity,
    COUNT(DISTINCT r.resevationId) as total_bookings,
    SUM(b.total_amount - b.discount_amount) as total_revenue,
    ROUND(SUM(b.total_amount - b.discount_amount) / f.capacity, 2) as revenue_per_capacity_unit,
    AVG(TIMESTAMPDIFF(HOUR,
        STR_TO_DATE(f.open_time, '%H:%i'),
        STR_TO_DATE(f.close_time, '%H:%i'))) as daily_operating_hours
FROM Facilities f
LEFT JOIN Reservation r ON f.facilityId = r.facilityId
LEFT JOIN Billing b ON r.resevationId = b.reservationId
WHERE b.billing_date >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
GROUP BY f.facilityId, f.facilitiy_name, f.facility_type, f.capacity
ORDER BY revenue_per_capacity_unit DESC;
```
