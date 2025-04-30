# Activitat Subprogrames

## Exercici 1 -  Funcions 
Exercici 1 - - Fes una funció anomenada spData, tal que donada una data en format MySQL ( AAAA-MM-DD ) ens retorni una cadena de caràcters en format DD-MM-AAAA. <br>
Exemple : SELECT spData('1988-12-01') => 01-12-1988

```mysql
DELIMITER //
DROP FUNCTION IF EXISTS sp_Data //
CREATE FUNCTION sp_Data(p_dataOrigen DATE) RETURNS VARCHAR(10) DETERMINISTIC
BEGIN
    DECLARE v_dataFinal VARCHAR(10);
    SET v_dataFinal = DATE_FORMAT(p_dataOrigen, '%d-%m-%Y');
    RETURN v_dataFinal;
END //
DELIMITER ;
SELECT rrhh.sp_Data('1988-12-01');
```

## Exercici 2 -  Funcions 
Exercici 2 - Fes una funció anomenada spPotencia, tal que donada una base i un exponent, ens calculi la seva potència. Intenta no utilitzar la funció POW. <br>
Exemple : SELECT spPotencia(2,3) => 8 

```mysql
DELIMITER //
DROP FUNCTION IF EXISTS sp_Potencia //
CREATE FUNCTION sp_Potencia (b INT, e INT) RETURNS BIGINT DETERMINISTIC
BEGIN
    DECLARE v_potencia BIGINT;
    IF b IS NOT NULL AND e IS NOT NULL THEN
        SET v_potencia = 1;
        WHILE e > 0 DO
            SET v_potencia = v_potencia * b;
            SET e = e - 1;
        END WHILE;
    END IF;
  
    RETURN v_potencia;
END //
DELIMITER ;
SELECT sp_Potencia(2, 3);
```

## Exercici 3 -  Funcions 
Exercici 3 - - Fes una funció anomenada spIncrement que donat un codi d’empleat i un % de increment, ens calculi el salari sumant aquest percentatge. <br>
Per exemple, suposem que l’ empleat amb id_empleat = 124 té un salari de 1000 <br>
Exemple: SELECT spIncrement(124,10) obtindriem -> 1100

```mysql
DELIMITER //
CREATE FUNCTION sp_Increment(id_empleat INT, increment_p INT)
RETURNS DECIMAL(10,2) DETERMINISTIC
BEGIN
    DECLARE salari_actual DECIMAL(10,2);
    
    SELECT salari INTO salari_actual
    FROM empleats
    WHERE id_empleat = id_empleat
    LIMIT 1;
    
    IF salari_actual IS NULL THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'No s''ha trobat cap empleat amb aquest id_empleat.';
    END IF;
    
    SET salari_actual = salari_actual + (salari_actual * (increment_p / 100));
    
    UPDATE empleats
    SET salari = salari_actual
    WHERE id_empleat = id_empleat;
    
    RETURN salari_actual;
END//
DELIMITER ;
SELECT rrhh.sp_Increment(212, 20);

```

## Exercici 4 -  Funcions 
Exercici 4 - Fes una funció anomenada spPringat, tal que li passem un codi de departament, i ens torni el codi d’empleat que guanya menys d’aquell departament. 

```mysql
DELIMITER //
CREATE FUNCTION sp_Pringat(codi_d INT)
RETURNS INT
NOT DETERMINISTIC READS SQL DATA
BEGIN
    DECLARE empleat_menys_pagat INT;
    
    SELECT empleat_id INTO empleat_menys_pagat
    FROM empleats
    WHERE codi_d = codi_d
    ORDER BY salari ASC
    LIMIT 1;
    
    RETURN empleat_menys_pagat;
END//
DELIMITER ;
SELECT rrhh.sp_Pringat(60);
```

## Exercici 5 -  Funcions 
Exercici 5 - Utilitzant la funció spPringat fes una consulta per obtenir de cada departament, l’empleat pringat. Mostra el codi i nom del departament, i el codi d’empleat.

```mysql
SELECT d.departament_id,sp_Pringat(d.departament_id) AS codi_empleat_pringat
FROM empleats d;
```

## Exercici 6 -  Funcions 
Exercici 6 - Fes una funció anomenada spCategoria, tal que donat un codi d’empleat, ens digui en quina categoria professional està. El criteri que volem seguir per determinar la categoria professional és en funció dels anys que porta treballant a l’empresa: <br>
Entre 0 i 1 anys -> Auxiliar <br>
Entre 2 i 10 anys -> Oficial de Segona <br>
Entre 11 i 20 Anys -> Oficial de Primera <br>
Més de 20 anys -> Que es jubili! <br>


```mysql
DROP FUNCTION IF EXISTS spCategoria;
DELIMITER //
CREATE FUNCTION spCategoria(codiEmpleat INT) RETURNS VARCHAR(50)
BEGIN
    DECLARE anys INT;
    DECLARE text_categoria VARCHAR(50);

    SELECT TIMESTAMPDIFF(YEAR, data_contractacio, CURRENT_DATE()) INTO anys
    FROM empleats
    WHERE empleat_id = codiEmpleat;

    IF anys >= 0 AND anys < 1 THEN
        SET text_categoria = 'Auxiliar';
    ELSEIF anys >= 2 AND anys <= 10 THEN
        SET text_categoria = 'Oficial de Segona';
    ELSEIF anys >= 11 AND anys <= 20 THEN
        SET text_categoria = 'Oficial de Primera';
    ELSE
        SET text_categoria = 'Que es jubili!';
    END IF;

    RETURN text_categoria;
END//
DELIMITER ;
 SELECT  spCategoria(empleat_id)
    FROM empleats

```

## Exercici 7 -  Funcions 
Exercici 7 - Fes una consulta utilitzant la funció anterior perquè mostri de cada empleat, el codi d’empleat, el nom, els anys treballats i la categoria professional a la que pertany.

```mysql
SELECT 
    e.codi_empleat,
    e.nom,
    TIMESTAMPDIFF(YEAR, e.data_ingres, CURDATE()) AS anys_treballats,
    spCategoria(e.codi_empleat) AS categoria_professional
FROM empleats e;

```

## Exercici 8 -  Funcions 
Exercici 8 - Fes una funció anomenada spEdat, tal que donada una data per paràmetre ens retorni l'edat d'una persona. Les dates posteriors a la data d'avui han de retornar 0.

```mysql
CREATE FUNCTION spEdat(data_naixement DATE) 
RETURNS INT
BEGIN
    DECLARE edat INT;
    
    -- Comprovem si la data de naixement és posterior a la data d'avui
    IF data_naixement > CURDATE() THEN
        RETURN 0;
    END IF;
    
    -- Calcula l'edat
    SET edat = TIMESTAMPDIFF(YEAR, data_naixement, CURDATE());
    
    -- Comprovem si la persona ja ha fet anys aquest any
    IF MONTH(data_naixement) > MONTH(CURDATE()) OR (MONTH(data_naixement) = MONTH(CURDATE()) AND DAY(data_naixement) > DAY(CURDATE())) THEN
        SET edat = edat - 1;
    END IF;
    
    RETURN edat;
END;

SELECT spEdat('1990-03-15') AS edat;

```

## Exercici 9 -  Funcions 
Exercici 9 - Fes una funció que ens retorni el número de directors (caps) diferents tenim.

```mysql
CREATE FUNCTION spDirectors() 
RETURNS INT
BEGIN
    DECLARE num_directors INT;
    
    -- Comptem els directors únics (aquells amb codi_director que no siguin NULL ni el mateix codi_empleat)
    SELECT COUNT(DISTINCT codi_director) INTO num_directors
    FROM empleats
    WHERE codi_director IS NOT NULL AND codi_empleat != codi_director;
    
    -- Retornem el nombre de directors
    RETURN num_directors;
END;

SELECT spDirectors() AS num_directors;

```

## Exercici 10 -  Funcions 
Exercici 10 - Quina instrucció utilitzarem si volem veure el contingut de la funció spPringat?

```mysql
SHOW CREATE FUNCTION spPringat;

SELECT pg_get_functiondef('spPringat'::regprocedure);

SELECT OBJECT_DEFINITION(OBJECT_ID('spPringat'));

```

***

## Exercici 1 - Procediments

Exercici 1 - Fes un procediment que permeti obtenir la data i hora del sistema i l’usuari actual. 

```mysql
DELIMITER $$

CREATE PROCEDURE obtenirDataHoraUsuari()
BEGIN
    -- Mostrem la data i hora actual
    SELECT NOW() AS data_hora_actual;

    -- Mostrem l'usuari actual
    SELECT USER() AS usuari_actual;
END$$

DELIMITER ;

CALL obtenirDataHoraUsuari();

```

## Exercici 2 - Procediments
Exercici 2 - Fes un procediment que intercanvii el sou de dos empleats passats per paràmetre.

```mysql
DELIMITER $$

CREATE PROCEDURE intercanviarSous(codi1 INT, codi2 INT)
BEGIN
    DECLARE sou1 DECIMAL(10, 2);
    DECLARE sou2 DECIMAL(10, 2);

    -- Obtenim els sous dels dos empleats
    SELECT sou INTO sou1 FROM empleats WHERE codi_empleat = codi1;
    SELECT sou INTO sou2 FROM empleats WHERE codi_empleat = codi2;

    -- Intercanviem els sous
    UPDATE empleats SET sou = sou2 WHERE codi_empleat = codi1;
    UPDATE empleats SET sou = sou1 WHERE codi_empleat = codi2;
END$$

DELIMITER ;

CALL intercanviarSous(1, 2);

```

## Exercici 3 - Procediments
Exercici 3 - Fes un procediment que donat dos Ids d'empleat assigni el codi de departament del primer en el segon.

```mysql
DELIMITER $$

CREATE PROCEDURE assignarDepartament(codi1 INT, codi2 INT)
BEGIN
    DECLARE departament1 INT;

    -- Obtenim el departament del primer empleat
    SELECT departament_id INTO departament1 FROM empleats WHERE codi_empleat = codi1;

    -- Assignem el departament del primer empleat al segon empleat
    UPDATE empleats SET departament_id = departament1 WHERE codi_empleat = codi2;
END$$

DELIMITER ;

CALL assignarDepartament(1, 2);

```

## Exercici 4 - Procediments
Exercici 4 - Fes un procediment que donat dos codis de departament assigni tots els empleats del segon en el primer. Un cop executat el procediment el departament que correspont en el segon paràmetre ha de quedar desert/sense cap empleat.

```mysql
DELIMITER $$

CREATE PROCEDURE transferirEmpleats(codi_departament1 INT, codi_departament2 INT)
BEGIN
    DECLARE empleats_count INT;

    -- Comprovem si el segon departament té empleats
    SELECT COUNT(*) INTO empleats_count FROM empleats WHERE departament_id = codi_departament2;

    IF empleats_count > 0 THEN
        -- Actualitzem els empleats del segon departament assignant-los al primer departament
        UPDATE empleats
        SET departament_id = codi_departament1
        WHERE departament_id = codi_departament2;

        -- Deixem el segon departament sense empleats
        UPDATE empleats
        SET departament_id = NULL
        WHERE departament_id = codi_departament2;
    ELSE
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El departament 2 ja està buit.';
    END IF;
END$$

DELIMITER ;

CALL transferirEmpleats(1, 2);

```

## Exercici 5 - Procediments
Exercici 5 - Fes un procediment per mostrar un llistat dels empleats. Volem veure el id_empleat, nom_empleat, nom_departament i el nom de la localització del departament.

```mysql
DELIMITER $$

CREATE PROCEDURE llistatEmpleats()
BEGIN
    SELECT 
        e.id_empleat,
        e.nom_empleat,
        d.nom_departament,
        l.nom_localitzacio
    FROM 
        empleats e
    JOIN 
        departaments d ON e.departament_id = d.departament_id
    JOIN 
        localitzacions l ON d.localitzacio_id = l.localitzacio_id;
END$$

DELIMITER ;

CALL llistatEmpleats();

```

## Exercici 6 - Procediments
Exercici 6 - Fes un procediment que donat un codi d’empleat, ens doni la informació de l’empleat ( agafa la informació que creguis rellevant).

```mysql
DELIMITER $$

CREATE PROCEDURE obtenirInformacioEmpleat(codi_empleat INT)
BEGIN
    -- Comprovem si l'empleat existeix
    IF NOT EXISTS (SELECT 1 FROM empleats WHERE id_empleat = codi_empleat) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'L\'empleat no existeix.';
    ELSE
        -- Si l'empleat existeix, obtenim la seva informació
        SELECT 
            e.id_empleat,
            e.nom_empleat,
            e.data_inici,
            e.sou,
            e.càrrec,
            d.nom_departament,
            l.nom_localitzacio
        FROM 
            empleats e
        JOIN 
            departaments d ON e.departament_id = d.departament_id
        JOIN 
            localitzacions l ON d.localitzacio_id = l.localitzacio_id
        WHERE 
            e.id_empleat = codi_empleat;
    END IF;
END$$

DELIMITER ;

CALL obtenirInformacioEmpleat(1);

```

## Exercici 7 - Procediments
Exercici 7 - Volem fer un registre dels usuaris que entren al nostre sistema. Per fer-ho primer caldrà crear una taula amb dos camps, un per guardar l’usuari i l’altre per guardar la data i hora de l’accés. 

```mysql
CREATE TABLE registres_entrades (
    usuari VARCHAR(255) NOT NULL,
    data_entrada DATETIME NOT NULL,
    PRIMARY KEY (usuari, data_entrada)
);


INSERT INTO registres_entrades (usuari, data_entrada)
VALUES ('johndoe', NOW());


DELIMITER $$

CREATE PROCEDURE enregistrarEntrada(usuari_iniciat VARCHAR(255))
BEGIN
    INSERT INTO registres_entrades (usuari, data_entrada)
    VALUES (usuari_iniciat, NOW());
END$$

DELIMITER ;


CALL enregistrarEntrada('janedoe');

```

## Exercici 8 - Procediments
Exercici 8 - A continuació feu un procediment sense arguments, de manera que cada vegada que el crideu, insereixi en aquesta taula l’usuari actual i la data i hora en que s’ha executat el procediment.

```mysql
DELIMITER $$

CREATE PROCEDURE registrarEntrada()
BEGIN
    INSERT INTO registres_entrades (usuari, data_entrada)
    VALUES (USER(), NOW());
END$$

DELIMITER ;

CALL registrarEntrada();

```

## Exercici 9 - Procediments
Exercici 9 - Fes un procediment que ens permeti afegir un nou departament però amb la següent particularitat: En cas que la localització no existeixi a la taula localitzacions, ens posarà un NULL en el camp id_localtizacio de la taula departaments. Al procediment li hem de passar el codi de departament, el nom del departament i el codi de la localització.

```mysql
DELIMITER $$

CREATE PROCEDURE afegirDepartament(
    codi_departament INT,
    nom_departament VARCHAR(255),
    codi_localitzacio INT
)
BEGIN
    DECLARE localitzacio_existent INT;

    -- Comprovem si la localització existeix a la taula localitzacions
    SELECT COUNT(*) INTO localitzacio_existent
    FROM localitzacions
    WHERE localitzacio_id = codi_localitzacio;

    -- Si la localització existeix, inserim amb el codi de la localització
    IF localitzacio_existent > 0 THEN
        INSERT INTO departaments (departament_id, nom_departament, id_localitzacio)
        VALUES (codi_departament, nom_departament, codi_localitzacio);
    ELSE
        -- Si la localització no existeix, inserim amb NULL en el camp id_localitzacio
        INSERT INTO departaments (departament_id, nom_departament, id_localitzacio)
        VALUES (codi_departament, nom_departament, NULL);
    END IF;
END$$

DELIMITER ;

CALL afegirDepartament(101, 'Recursos Humans', 5);

```
## Exercici 10 - Procediments
Exercici 10 - Fes un procediment que donat un codi d’empleat, ens posi en paràmetres de sortida el nom i el cognom. Indica com ho pots fer per comprovar si el procediment et funciona.

```mysql
DELIMITER $$

CREATE PROCEDURE obtenirNomCognom(
    IN codi_empleat INT,     -- Paràmetre d'entrada: codi d'empleat
    OUT nom_empleat VARCHAR(255),  -- Paràmetre de sortida: nom de l'empleat
    OUT cognom_empleat VARCHAR(255)  -- Paràmetre de sortida: cognom de l'empleat
)
BEGIN
    -- Consulta per obtenir el nom i cognom de l'empleat a partir del codi
    SELECT nom, cognom
    INTO nom_empleat, cognom_empleat
    FROM empleats
    WHERE id_empleat = codi_empleat;
END$$

DELIMITER ;

CALL obtenirNomCognom(123, @nom, @cognom);

```

## Exercici 11 - Procediments
Exercici 11 - Fes un procediment que ens permeti modificar el nom i cognom d’un empleat. 

```mysql
DELIMITER $$

CREATE PROCEDURE modificarNomCognom(
    IN codi_empleat INT,        -- Codi de l'empleat a modificar
    IN nou_nom VARCHAR(255),    -- Nou nom de l'empleat
    IN nou_cognom VARCHAR(255)  -- Nou cognom de l'empleat
)
BEGIN
    -- Actualització del nom i cognom de l'empleat
    UPDATE empleats
    SET nom = nou_nom, cognom = nou_cognom
    WHERE id_empleat = codi_empleat;
END$$

DELIMITER ;

CALL modificarNomCognom(123, 'Joan', 'Pérez');

```

## Exercici 12 - Procediments
Exercici 12 - Crea una taula d’auditoria anomenada logs_usuaris. Aquesta taula la utilitzarem per monitoritzar algunes de les accions que fan els usuaris sobre les dades,per exemple si actualitzen dades, eliminen registres.
La taula ha de tenir els següents camps:

| Nom | Tipus | Descripció |
| --- | --- | --- |
| usuari | VARCHAR(100)  | Usuari que ha realitzat l’acció |
| data   | DATETIME      | Data-Hora en que s’ha realitzat l’acció |
| taula  | VARCHAR(50)   | Taula sobre la que es realitza l’acció |
| accio  | VARCHAR(20)   | “ELIMINAR”,”AFEGIR”,”MODIFICAR”,”INSERIR” |
| valor_pk | VARCHAR(200) | valor que identifica el registre que ha patit l’acció |

Fes un procediment amb nom spRegistrarLog que rebrà com a paràmetres el nom de la taula, l’acció i el valor_pk.
Aquest procediment només cal que insereixi un registre en la taula logs_usuaris amb les dades rebudes, tenint en compte l’usuari actual i la data-hora del sistema.

```mysql
CREATE TABLE logs_usuaris (
    id INT AUTO_INCREMENT PRIMARY KEY,    
    usuari VARCHAR(100),                  
    data DATETIME,                        
    taula VARCHAR(50),                    
    accio VARCHAR(20),                     
    valor_pk VARCHAR(200)                  
);

DELIMITER $$

CREATE PROCEDURE spRegistrarLog(
    IN taula_name VARCHAR(50),           -- Nom de la taula
    IN accio_type VARCHAR(20),           -- Tipus d'acció (ELIMINAR, AFEGIR, MODIFICAR, INSERIR)
    IN valor_pk VARCHAR(200)            -- Valor de la clau primària del registre afectat
)
BEGIN
    DECLARE current_user VARCHAR(100);
    DECLARE current_datetime DATETIME;

    -- Obtenim l'usuari actual i la data-hora del sistema
    SET current_user = USER();              -- Obtenció de l'usuari actual
    SET current_datetime = NOW();           -- Obtenció de la data-hora del sistema

    -- Inserim el registre a la taula logs_usuaris
    INSERT INTO logs_usuaris (usuari, data, taula, accio, valor_pk)
    VALUES (current_user, current_datetime, taula_name, accio_type, valor_pk);
END$$

DELIMITER ;

CALL spRegistrarLog('empleats', 'ELIMINAR', '123');

```

## Exercici 13 - Procediments
Exercici 13 - Fes un procediment que ens permeti eliminar un departament determinat. El departament s’ha d’eliminar de la taula departaments. Utilitza a més dins d’aquest procediment, el procediment creat anteriorment (spRegistrarLog) per guardar també un registre del que ha realitzat l’usuari. Ens ha de quedar clar que l’usuari actual, en data X ha eliminat de la taula DEPARTAMENTS el codi departament Y. <br>
Per exemple, si fem una crida al procediment per eliminar un departament:<br>
CALL eliminarDept(300);

```mysql
DELIMITER $$

CREATE PROCEDURE eliminarDept(
    IN codi_dept INT              -- Codi del departament a eliminar
)
BEGIN
    -- Eliminar el departament de la taula departaments
    DELETE FROM departaments
    WHERE id_departament = codi_dept;

    -- Registrar l'eliminació a la taula de logs_usuaris
    CALL spRegistrarLog('DEPARTAMENTS', 'ELIMINAR', codi_dept);
END$$

DELIMITER ;

CALL eliminarDept(300);

```

## Exercici 14 - Procediments
Exercici 14 - Fes un procediment que ens posi en paràmetres de sortida, el número d’empleats que tenim, el número de departaments i el número de localitzacions.

```mysql
DELIMITER $$

CREATE PROCEDURE spContarEmpleatsDepartamentsLocalitzacions(
    OUT num_empleats INT,            
    OUT num_departaments INT,        
    OUT num_localitzacions INT       
)
BEGIN
    -- Comptem el nombre total d'empleats
    SELECT COUNT(*) INTO num_empleats FROM empleats;

    -- Comptem el nombre total de departaments
    SELECT COUNT(*) INTO num_departaments FROM departaments;

    -- Comptem el nombre total de localitzacions
    SELECT COUNT(*) INTO num_localitzacions FROM localitzacions;
END$$

DELIMITER ;

CALL spContarEmpleatsDepartamentsLocalitzacions(@num_empleats, @num_departaments, @num_localitzacions);

```
