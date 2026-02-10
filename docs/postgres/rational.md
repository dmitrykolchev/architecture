## PostgreSQL rational data type

``` sql

DROP TYPE IF EXISTS rational CASCADE;


-- Создаем составной тип
CREATE TYPE rational AS (
    i 	NUMERIC(28, 0),
    n   BIGINT,
    d  	BIGINT
);

CREATE OR REPLACE FUNCTION gcd(a BIGINT, b BIGINT)
RETURNS BIGINT AS $$
BEGIN
    WHILE b <> 0 LOOP
        a := a % b;
        -- Обмен a и b
        a := a + b;
        b := a - b;
        a := a - b;
    END LOOP;
    RETURN a;
END;
$$ LANGUAGE plpgsql IMMUTABLE;


-- Функция для нормализации (приведения к каноническому виду)
-- Она будет сокращать дробь и переносить "целую" часть из дроби в integer_part
CREATE OR REPLACE FUNCTION rationalize(num rational)
RETURNS rational AS $$
DECLARE
    common_divisor BIGINT;
    int_from_frac BIGINT;
BEGIN
    -- 1. Обработка нулевого знаменателя (ошибка)
    IF num.d = 0 THEN
        RAISE EXCEPTION 'Denominator cannot be zero';
    END IF;

    -- 2. Переносим знак в числитель
    IF num.d < 0 THEN
        num.n := -num.n;
        num.d := -num.d;
    END IF;

    -- 3. Выделяем целую часть из неправильной дроби
    int_from_frac := num.n / num.d;
    num.i := num.i + int_from_frac;
    num.n := num.n % num.d;

    -- 4. Сокращаем дробь
    IF num.n <> 0 THEN
        common_divisor := gcd(abs(num.n), num.d);
        num.n := num.n / common_divisor;
        num.d := num.d / common_divisor;
    END IF;
    
    -- 5. Если числитель стал 0, знаменатель ставим 1 для каноничности
    IF num.n = 0 THEN
        num.d := 1;
    END IF;

    RETURN num;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE OR REPLACE FUNCTION rational_from(i NUMERIC, n BIGINT, d BIGINT)
RETURNS rational AS $$
DECLARE
    r rational;
BEGIN
    r := ROW(i, n, d)::rational;
    RETURN rationalize(r);
END;
$$ LANGUAGE plpgsql IMMUTABLE; -- Changed from sql to plpgsql

-- Функция ввода из текста (например, '123 4/5' или '123.45')
CREATE OR REPLACE FUNCTION rational_in(text_val TEXT)
RETURNS rational AS $$
DECLARE
    int_part_text TEXT;
    frac_part_text TEXT;
    space_pos INTEGER;
    dot_pos INTEGER;
    slash_pos INTEGER;
    int_val NUMERIC(28, 0) := 0;
    num_val BIGINT := 0;
    den_val BIGINT := 1;
BEGIN
    -- Парсим смешанную дробь типа "123 4/5"
    space_pos := strpos(text_val, ' ');
    dot_pos := strpos(text_val, '.');

    IF space_pos > 0 THEN -- Формат "123 4/5"
        int_part_text := trim(substring(text_val for space_pos));
        frac_part_text := trim(substring(text_val from space_pos));
        slash_pos := strpos(frac_part_text, '/');
        
        int_val := int_part_text::NUMERIC(28, 0);
        num_val := substring(frac_part_text for slash_pos - 1)::BIGINT;
        den_val := substring(frac_part_text from slash_pos + 1)::BIGINT;

    ELSIF dot_pos > 0 THEN -- Формат "123.45"
        int_part_text := substring(text_val for dot_pos - 1);
        frac_part_text := substring(text_val from dot_pos + 1);

        int_val := int_part_text::NUMERIC(28, 0);
        num_val := frac_part_text::BIGINT;
        den_val := (10^length(frac_part_text))::BIGINT;
    
    ELSE -- Формат "123"
        int_val := text_val::NUMERIC(28, 0);
        num_val := 0;
        den_val := 1;
    END IF;

    RETURN rationalize(ROW(int_val, num_val, den_val)::rational);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Функция вывода в виде текста
CREATE OR REPLACE FUNCTION rational_out(val rational)
RETURNS TEXT AS $$
BEGIN
    IF val.n = 0 THEN
        RETURN val.i::TEXT;
    ELSE
        RETURN val.i::TEXT || ' ' || val.n::TEXT || '/' || val.d::TEXT;
    END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;


-- Внутренняя функция для преобразования в NUMERIC для операций
CREATE OR REPLACE FUNCTION rational_to_numeric(val rational)
RETURNS NUMERIC AS $$
BEGIN
    RETURN val.i + (val.n::NUMERIC / val.d::NUMERIC);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Функция сравнения
CREATE OR REPLACE FUNCTION rational_cmp(a rational, b rational)
RETURNS INTEGER AS $$
DECLARE
    a_numeric NUMERIC := rational_to_numeric(a);
    b_numeric NUMERIC := rational_to_numeric(b);
BEGIN
    IF a_numeric > b_numeric THEN RETURN 1;
    ELSIF a_numeric < b_numeric THEN RETURN -1;
    ELSE RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Step 3: Create helper functions (one for each operator)
CREATE OR REPLACE FUNCTION rational_eq(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) = 0; $$ LANGUAGE sql IMMUTABLE;
CREATE OR REPLACE FUNCTION rational_ne(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) <> 0; $$ LANGUAGE sql IMMUTABLE;
CREATE OR REPLACE FUNCTION rational_lt(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) = -1; $$ LANGUAGE sql IMMUTABLE;
CREATE OR REPLACE FUNCTION rational_le(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) <= 0; $$ LANGUAGE sql IMMUTABLE;
CREATE OR REPLACE FUNCTION rational_gt(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) = 1; $$ LANGUAGE sql IMMUTABLE;
CREATE OR REPLACE FUNCTION rational_ge(a rational, b rational) RETURNS BOOLEAN AS $$ SELECT rational_cmp(a, b) >= 0; $$ LANGUAGE sql IMMUTABLE;

-- Step 4: Create the operators themselves
-- Use DROP OPERATOR...CASCADE to handle dependencies if you're re-running the script
DROP OPERATOR IF EXISTS = (rational, rational) CASCADE;
CREATE OPERATOR = (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_eq,
    COMMUTATOR = =,
    NEGATOR = <>
);

DROP OPERATOR IF EXISTS <> (rational, rational) CASCADE;
CREATE OPERATOR <> (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_ne,
    COMMUTATOR = <>,
    NEGATOR = =
);

DROP OPERATOR IF EXISTS < (rational, rational) CASCADE;
CREATE OPERATOR < (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_lt,
    COMMUTATOR = >,
    NEGATOR = >=
);

DROP OPERATOR IF EXISTS <= (rational, rational) CASCADE;
CREATE OPERATOR <= (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_le,
    COMMUTATOR = >=,
    NEGATOR = >
);

DROP OPERATOR IF EXISTS > (rational, rational) CASCADE;
CREATE OPERATOR > (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_gt,
    COMMUTATOR = <,
    NEGATOR = <=
);

DROP OPERATOR IF EXISTS >= (rational, rational) CASCADE;
CREATE OPERATOR >= (
    LEFTARG = rational,
    RIGHTARG = rational,
    PROCEDURE = rational_ge,
    COMMUTATOR = <=,
    NEGATOR = <
);
-- Создаем B-Tree операторный класс для индексации
CREATE OPERATOR CLASS rational_opc
DEFAULT FOR TYPE rational USING btree AS
    OPERATOR 1 < (rational, rational),
    OPERATOR 2 <= (rational, rational),
    OPERATOR 3 = (rational, rational),
    OPERATOR 4 >= (rational, rational),
    OPERATOR 5 > (rational, rational),
    FUNCTION 1 rational_cmp(rational, rational);

-- Функция сложения двух рациональных чисел
CREATE OR REPLACE FUNCTION rational_add(a rational, b rational)
RETURNS rational AS $$
DECLARE
    new_num BIGINT;
    new_den BIGINT;
    result rational;
BEGIN
    -- Приводим к общему знаменателю
    new_den := a.d * b.d;
    new_num := a.n * b.d + b.n * a.d;

    result := ROW(a.i + b.i, new_num, new_den)::rational;
    RETURN rationalize(result);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Создаем агрегат SUM
CREATE AGGREGATE SUM(rational) (
    SFUNC = rational_add,
    STYPE = rational,
    INITCOND = '(0, 0, 1)'
);	

-- CAST в NUMERIC у нас уже есть в виде функции rational_to_numeric
CREATE CAST (rational AS NUMERIC)
WITH FUNCTION rational_to_numeric(rational) AS IMPLICIT;

-- CAST из NUMERIC
CREATE OR REPLACE FUNCTION numeric_to_rational(val NUMERIC)
RETURNS rational AS $$
    SELECT rational_in(val::TEXT);
$$ LANGUAGE sql IMMUTABLE;
CREATE CAST (NUMERIC AS rational)
WITH FUNCTION numeric_to_rational(NUMERIC) AS ASSIGNMENT;


CREATE TABLE products (
    name TEXT,
    weight rational
);

DROP CAST IF EXISTS (text AS rational);

-- Create the cast
CREATE CAST (text AS rational)
WITH FUNCTION rational_in(text)
AS IMPLICIT;

-- The new INPUT function must take `cstring`
CREATE OR REPLACE FUNCTION rational_in_from_cstring(val cstring)
RETURNS rational AS $$
DECLARE
    -- Cast cstring to text to use string functions
    text_val TEXT := val;
    int_part_text TEXT;
    frac_part_text TEXT;
    space_pos INTEGER;
    dot_pos INTEGER;
    slash_pos INTEGER;
    int_val NUMERIC(28, 0) := 0;
    num_val BIGINT := 0;
    den_val BIGINT := 1;
BEGIN
    -- The rest of your parsing logic remains exactly the same!
    dot_pos := strpos(text_val, '.');
    space_pos := strpos(text_val, ' ');

    IF dot_pos > 0 THEN -- Format "123.45"
        int_part_text := substring(text_val from 1 for dot_pos - 1);
        frac_part_text := substring(text_val from dot_pos + 1);
        int_val := int_part_text::NUMERIC(28, 0);
        num_val := frac_part_text::BIGINT;
        den_val := (10^length(frac_part_text))::BIGINT;

    ELSIF space_pos > 0 THEN -- Format "123 4/5"
        int_part_text := trim(substring(text_val for space_pos));
        frac_part_text := trim(substring(text_val from space_pos));
        slash_pos := strpos(frac_part_text, '/');
        int_val := int_part_text::NUMERIC(28, 0);
        num_val := substring(frac_part_text for slash_pos - 1)::BIGINT;
        den_val := substring(frac_part_text from slash_pos + 1)::BIGINT;
    
    ELSE -- Format "123"
        int_val := text_val::NUMERIC(28, 0);
        num_val := 0;
        den_val := 1;
    END IF;

    RETURN rationalize(ROW(int_val, num_val, den_val)::rational);
END;
$$ LANGUAGE plpgsql IMMUTABLE;


-- The new OUTPUT function must return `cstring`
CREATE OR REPLACE FUNCTION rational_out_to_cstring(val rational)
RETURNS cstring AS $$
DECLARE
    result_text TEXT;
BEGIN
    IF val.numerator = 0 THEN
        result_text := val.integer_part::TEXT;
    ELSE
        result_text := val.integer_part::TEXT || ' ' || val.numerator::TEXT || '/' || val.denominator::TEXT;
    END IF;
    RETURN result_text::cstring;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

DROP CAST IF EXISTS (numeric AS rational);
CREATE CAST (numeric AS rational)
WITH FUNCTION rational(text) -- It will implicitly convert numeric->text first
AS ASSIGNMENT;

DROP CAST IF EXISTS (integer AS rational);
CREATE CAST (integer AS rational)
WITH FUNCTION rational(text)
AS ASSIGNMENT;

CREATE TABLE IF NOT EXISTS public.products
(
    name text COLLATE pg_catalog."default",
    weight rational
)

INSERT INTO products VALUES
('Flour', rational('1.5')),          -- 1 1/2
('Sugar', rational('0 900/1000')),   -- 0 9/10
('Salt', rational('1')),
('Butter', rational('0.25'));        -- 0 1/4

select as_text(sum(weight)) from products

select cast('1.5' as rational)

select rational_out(rational('1.5'))
```
