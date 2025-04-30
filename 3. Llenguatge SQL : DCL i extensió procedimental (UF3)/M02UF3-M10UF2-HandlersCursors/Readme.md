# Activitat Handlers i Cursors

**Exercici 1**

```mysql
DELIMITER $$

CREATE PROCEDURE copia_empleat(IN codi_empleat VARCHAR(100))
BEGIN
    DECLARE empleat_existent INT;

    -- Verificar si l'empleat existeix a la taula empleats
    SELECT COUNT(*) INTO empleat_existent
    FROM empleats
    WHERE id_empleat = codi_empleat;

    -- Si la taula empleat_copia no existeix, la creem
    CREATE TABLE IF NOT EXISTS empleat_copia (
        id_empleat VARCHAR(100),
        nom VARCHAR(100),
        cognoms VARCHAR(100)
    );

    -- Si l'empleat existeix, fem la còpia
    IF empleat_existent > 0 THEN
        -- Inserim a la taula empleat_copia
        INSERT INTO empleat_copia (id_empleat, nom, cognoms)
        SELECT id_empleat, nom, cognoms
        FROM empleats
        WHERE id_empleat = codi_empleat;
        
        -- Registrem la còpia a la taula logs_usuaris
        INSERT INTO logs_usuaris (usuari, data, taula, accio, valor_pk, error)
        VALUES (USER(), NOW(), 'empleat_copia', 'COPIA_EMPL', codi_empleat, 0);

    ELSE
        -- Si l'empleat no existeix, registrem un error a logs_usuaris
        INSERT INTO logs_usuaris (usuari, data, taula, accio, valor_pk, error)
        VALUES (USER(), NOW(), 'empleat_copia', 'COPIA_EMPL', codi_empleat, 1);
    END IF;

    -- Comprovem si ja existeix el registre a empleat_copia abans de fer la inserció
    IF EXISTS (SELECT 1 FROM empleat_copia WHERE id_empleat = codi_empleat) THEN
        -- Si l'empleat ja existeix a empleat_copia, registrem un error
        INSERT INTO logs_usuaris (usuari, data, taula, accio, valor_pk, error)
        VALUES (USER(), NOW(), 'empleat_copia', 'COPIA_EMPL', codi_empleat, 2);
    END IF;

END $$

DELIMITER ;

CALL copia_empleat('1234');

```

**Exercici 2**

```mysql
CREATE TABLE categories (
    codi CHAR(2) PRIMARY KEY,
    nom VARCHAR(30),
    quantitat SMALLINT UNSIGNED
);

DELIMITER $$

CREATE FUNCTION sp_Categoria(nom_categoria VARCHAR(30)) 
RETURNS CHAR(2)
BEGIN
    DECLARE codi_categoria CHAR(2);

    CASE
        WHEN nom_categoria = 'Auxiliar' THEN
            SET codi_categoria = 'C1';
        WHEN nom_categoria = 'Oficial de Segona' THEN
            SET codi_categoria = 'C2';
        WHEN nom_categoria = 'Oficial de Primera' THEN
            SET codi_categoria = 'C3';
        WHEN nom_categoria = 'Que es jubili!' THEN
            SET codi_categoria = 'C4';
        ELSE
            SET codi_categoria = NULL;
    END CASE;

    RETURN codi_categoria;
END $$

DELIMITER ;

DELIMITER $$

CREATE PROCEDURE comptabilitza_empleats_per_categoria()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE categoria_actual CHAR(2);
    DECLARE nombre_empleats INT;

    -- Declarar el cursor per recórrer les categories
    DECLARE categoria_cursor CURSOR FOR
        SELECT DISTINCT categoria
        FROM empleats;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Obrim el cursor
    OPEN categoria_cursor;

    -- Bucle per processar cada categoria
    read_loop: LOOP
        -- Obtenim el codi de la categoria
        FETCH categoria_cursor INTO categoria_actual;
        
        -- Si no hi ha més categories, sortim del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Comptabilitzem el nombre d'empleats en aquesta categoria
        SELECT COUNT(*) INTO nombre_empleats
        FROM empleats
        WHERE categoria = categoria_actual;

        -- Actualitzem la taula categories amb el nombre d'empleats per categoria
        UPDATE categories
        SET quantitat = nombre_empleats
        WHERE codi = categoria_actual;

    END LOOP;

    -- Tanquem el cursor
    CLOSE categoria_cursor;
END $$

DELIMITER ;

CALL comptabilitza_empleats_per_categoria();

```

**Exercici 3**

```mysql
ALTER TABLE departaments
ADD COLUMN salari_avg DECIMAL(10, 2);

DELIMITER $$

CREATE PROCEDURE actualitza_salari_avg()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE departament_id INT;
    DECLARE salari_total DECIMAL(10, 2);
    DECLARE nombre_empleats INT;
    DECLARE salari_mitja DECIMAL(10, 2);

    -- Declarar el cursor per recórrer els departaments
    DECLARE departament_cursor CURSOR FOR
        SELECT id_departament
        FROM departaments;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Obrim el cursor
    OPEN departament_cursor;

    -- Bucle per processar cada departament
    read_loop: LOOP
        -- Obtenim l'ID del departament
        FETCH departament_cursor INTO departament_id;
        
        -- Si no hi ha més departaments, sortim del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Comptabilitzem el salari total i el nombre d'empleats en aquest departament
        SELECT SUM(salari), COUNT(*)
        INTO salari_total, nombre_empleats
        FROM empleats
        WHERE departament_id = departament_id;

        -- Calculem la mitjana de salaris
        IF nombre_empleats > 0 THEN
            SET salari_mitja = salari_total / nombre_empleats;
        ELSE
            SET salari_mitja = 0; -- Si no hi ha empleats, la mitjana serà 0
        END IF;

        -- Actualitzem la taula departaments amb la mitjana de salaris
        UPDATE departaments
        SET salari_avg = salari_mitja
        WHERE id_departament = departament_id;

    END LOOP;

    -- Tanquem el cursor
    CLOSE departament_cursor;
END $$

DELIMITER ;

CALL actualitza_salari_avg();

```

**Exercici 4**

```mysql
CREATE TABLE pringats (
    empleat_id INT,
    departament_id INT,
    CONSTRAINT PK_PRINGATS PRIMARY KEY (departament_id, empleat_id)
);

DELIMITER $$

CREATE PROCEDURE omplir_pringats()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE departament_id INT;
    DECLARE empleat_id INT;
    DECLARE salari_min DECIMAL(10, 2);
    
    -- Declarar el cursor per recórrer els departaments
    DECLARE departament_cursor CURSOR FOR
        SELECT DISTINCT departament_id
        FROM empleats;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Esborrar el contingut de la taula pringats
    TRUNCATE TABLE pringats;

    -- Obrim el cursor
    OPEN departament_cursor;

    -- Bucle per processar cada departament
    read_loop: LOOP
        -- Obtenim l'ID del departament
        FETCH departament_cursor INTO departament_id;
        
        -- Si no hi ha més departaments, sortim del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Trobar l'empleat amb el salari més baix en aquest departament (pringat)
        SELECT empleat_id, MIN(salari) INTO empleat_id, salari_min
        FROM empleats
        WHERE departament_id = departament_id
        GROUP BY departament_id;

        -- Inserir el "pringat" a la taula pringats
        INSERT INTO pringats (empleat_id, departament_id)
        VALUES (empleat_id, departament_id);

    END LOOP;

    -- Tanquem el cursor
    CLOSE departament_cursor;
END $$

DELIMITER ;

CALL omplir_pringats();

```

**Exercici 5**

```mysql
DELIMITER $$

CREATE PROCEDURE omplir_pringats_sense_duplicats()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE departament_id INT;
    DECLARE empleat_id INT;
    DECLARE salari_min DECIMAL(10, 2);
    
    -- Declarar el cursor per recórrer els departaments
    DECLARE departament_cursor CURSOR FOR
        SELECT DISTINCT departament_id
        FROM empleats;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Esborrar el contingut de la taula pringats
    TRUNCATE TABLE pringats;

    -- Obrim el cursor
    OPEN departament_cursor;

    -- Bucle per processar cada departament
    read_loop: LOOP
        -- Obtenim l'ID del departament
        FETCH departament_cursor INTO departament_id;
        
        -- Si no hi ha més departaments, sortim del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Trobar l'empleat amb el salari més baix en aquest departament (pringat)
        SELECT empleat_id, MIN(salari) INTO empleat_id, salari_min
        FROM empleats
        WHERE departament_id = departament_id
        GROUP BY departament_id;

        -- Comprovar si el "pringat" ja existeix a la taula pringats
        IF NOT EXISTS (
            SELECT 1
            FROM pringats
            WHERE empleat_id = empleat_id AND departament_id = departament_id
        ) THEN
            -- Inserir el "pringat" a la taula pringats només si no existeix
            INSERT INTO pringats (empleat_id, departament_id)
            VALUES (empleat_id, departament_id);
        END IF;

    END LOOP;

    -- Tanquem el cursor
    CLOSE departament_cursor;
END $$

DELIMITER ;

CALL omplir_pringats_sense_duplicats();

```

**Exercici 6**

```mysql
CREATE TABLE empleats_segregats (
    id_empleat INT,
    mes_contratacio INT,
    CONSTRAINT PK_EMPLEATS_SEGREGATS PRIMARY KEY (id_empleat)
);

DELIMITER $$

CREATE PROCEDURE segregar_empleats_per_mes()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE id_emp INT;
    DECLARE mes_contratacio INT;

    -- Declarar el cursor per recórrer els empleats
    DECLARE empleat_cursor CURSOR FOR
        SELECT id_empleat, MONTH(data_contratacio) AS mes_contratacio
        FROM empleats;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Obrir el cursor
    OPEN empleat_cursor;

    -- Bucle per inserir a la taula empleats_segregats
    read_loop: LOOP
        -- Obtenir dades del cursor
        FETCH empleat_cursor INTO id_emp, mes_contratacio;
        
        -- Si no hi ha més dades, sortir del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Inserir a la taula empleats_segregats
        INSERT INTO empleats_segregats (id_empleat, mes_contratacio)
        VALUES (id_emp, mes_contratacio);

    END LOOP;

    -- Tancar el cursor
    CLOSE empleat_cursor;
END $$

DELIMITER ;

CALL segregar_empleats_per_mes();

```

**Exercici 7**

```mysql
ALTER TABLE FEINES
ADD COLUMN num_treballadors INT DEFAULT 0;

DELIMITER $$

CREATE PROCEDURE omplir_num_treballadors_feines()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE feina_id INT;
    DECLARE num_empleats INT;

    -- Declarar el cursor per recórrer les feines disponibles
    DECLARE feina_cursor CURSOR FOR
        SELECT id_feina
        FROM FEINES;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Obrir el cursor per recórrer les feines
    OPEN feina_cursor;

    -- Bucle per actualitzar cada perfil de feina amb el nombre d'empleats
    read_loop: LOOP
        -- Obtenir l'ID de la feina
        FETCH feina_cursor INTO feina_id;
        
        -- Si no hi ha més feines, sortir del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Comptar el nombre d'empleats que han treballat en aquest perfil
        SELECT COUNT(DISTINCT empleat_id) INTO num_empleats
        FROM empleats
        WHERE id_feina = feina_id AND (data_fi IS NULL OR data_fi >= CURDATE());

        -- Actualitzar la taula FEINES amb el nombre d'empleats per aquest perfil
        UPDATE FEINES
        SET num_treballadors = num_empleats
        WHERE id_feina = feina_id;

    END LOOP;

    -- Tancar el cursor
    CLOSE feina_cursor;
END $$

DELIMITER ;

CALL omplir_num_treballadors_feines();

```

**Exercici 8**

```mysql
DELIMITER $$

CREATE PROCEDURE recalcular_import_factures()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE factura_id INT;
    DECLARE import_total DECIMAL(8,2);
    DECLARE linia_import DECIMAL(8,2);

    -- Declarar el cursor per recórrer les factures
    DECLARE factura_cursor CURSOR FOR
        SELECT numf
        FROM factura;

    -- Declarar un handler per controlar quan el cursor arriba al final
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    -- Obrir el cursor per recórrer les factures
    OPEN factura_cursor;

    -- Bucle per calcular i actualitzar els imports de les factures
    read_loop: LOOP
        -- Obtenir el numf de la factura
        FETCH factura_cursor INTO factura_id;

        -- Si no hi ha més factures, sortir del bucle
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Inicialitzar l'import total per a la factura
        SET import_total = 0;

        -- Calcular l'import total de les línies associades a aquesta factura
        DECLARE linia_cursor CURSOR FOR
            SELECT unitats * preuU
            FROM linia_factura
            WHERE numf = factura_id;

        -- Obrir el cursor per recórrer les línies de la factura
        OPEN linia_cursor;

        -- Bucle per sumar els imports de les línies
        linia_loop: LOOP
            FETCH linia_cursor INTO linia_import;
            IF done THEN
                LEAVE linia_loop;
            END IF;

            -- Afegir el valor de la línia al total de la factura
            SET import_total = import_total + linia_import;
        END LOOP;

        -- Tancar el cursor de les línies de la factura
        CLOSE linia_cursor;

        -- Actualitzar el camp import de la taula factura amb l'import calculat
        UPDATE factura
        SET import = import_total
        WHERE numf = factura_id;

    END LOOP;

    -- Tancar el cursor de les factures
    CLOSE factura_cursor;
END $$

DELIMITER ;

CALL recalcular_import_factures();

```
