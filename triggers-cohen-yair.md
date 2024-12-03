1. Trigger de Mise à Jour de la Disponibilité des Voitures lors d'une Réservation
   Nom du Trigger : update_voiture_disponibilite
   Événement : AFTER INSERT sur la table Réservations
   Objectif :Lorsqu'un client effectue une réservation, la disponibilité de la voiture doit être mise à jour

Code SQL :
DELIMITER //

CREATE TRIGGER update_voiture_disponibilite
AFTER INSERT
ON Réservations
FOR EACH ROW
BEGIN
UPDATE app_db.Voitures
SET disponible = 0
WHERE id = NEW.voiture_id;
END //

2. Trigger de Vérification de l'Âge du Client avant une Réservation
      Nom du Trigger : verif_age_client
      Événement : BEFORE INSERT sur la table Réservations
      Objectif :L'age minimum pour louer une voiture est de 21 ans

Code SQL :
DELIMITER //

CREATE TRIGGER verif_age_client
BEFORE INSERT
ON Réservations
FOR EACH ROW
BEGIN
DECLARE client_age INT;

SELECT age INTO client_age FROM Clients WHERE id = NEW.client_id;

IF client_age < 21 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'L\'âge du client doit être d\'au moins 21 ans pour effectuer une réservation.';
    END IF;
END //

3. Trigger de Vérification de la Validité du Permis de Conduire avant la création d'un nouveau client
   Nom du Trigger : verif_validite_permis
   Événement : BEFORE INSERT sur la table Clients
   Objectif :Un numéro de permis de conduire doit être unique et est considére valide s'il a une longueur de 15 caractères alphanumériques (chiffres et lettres)
   SQl propose une fonction LENGTH() qui permet de calculer la longueur d'une chaîne de caractères et une fonction REGEXP pour vérifier si une chaîne de caractères correspond à un motif donné ( exemple : REGEXP '^[A-Z0-9]{8,12}$')
   Si le client n'a pas de permis de conduire, il ne peut pas être ajouté à la base de données et un message d'erreur doit être renvoyé

code SQL :
CREATE TRIGGER verif_validite_permis
BEFORE INSERT
ON Clients
FOR EACH ROW
BEGIN
IF NEW.permis_conduire IS NULL OR NEW.permis_conduire = '' THEN
SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'Un numéro de permis de conduire valide est requis pour l\'inscription.';
END IF;

IF LENGTH(NEW.permis_conduire) != 15 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Le numéro de permis de conduire doit avoir exactement 15 caractères.';
    END IF;

IF NEW.permis_conduire NOT REGEXP '^[A-Za-z0-9]{15}$' THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Le numéro de permis de conduire doit être alphanumérique et avoir 15 caractères.';
    END IF;

IF EXISTS (SELECT 1 FROM Clients WHERE permis_conduire = NEW.permis_conduire) THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Ce numéro de permis de conduire est déjà utilisé par un autre client.';
    END IF;
END //

4.  Trigger de Vérification de la Disponibilité de la Voiture avant une Réservation
   Nom du Trigger : verif_voiture_disponibilite
   Événement : BEFORE INSERT sur la table Réservations
   Objectif :Une voiture ne peut pas être réservée si elle est n'est pas disponible

code SQL :
DELIMITER //

CREATE TRIGGER verif_voiture_disponibilite
BEFORE INSERT
ON Réservations
FOR EACH ROW
BEGIN
DECLARE voiture_disponible BOOLEAN;

SELECT disponible INTO voiture_disponible
    FROM Voitures
    WHERE id = NEW.voiture_id;

IF voiture_disponible = FALSE THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'La voiture sélectionnée n\'est pas disponible pour la réservation.';
    END IF;
END //


5. Trigger pour Éviter les Chevauchements de Réservations sur la Même Voiture
   Nom du Trigger : check_reservation_overlap
   Événement : BEFORE INSERT sur la table Réservations
   Objectif :S'assurer qu'une voiture ne soit réservée par plusieurs clients pour des périodes qui se chevauchent.
   Modalités de rendu

code SQL :
DELIMITER //

CREATE TRIGGER check_reservation_overlap
BEFORE INSERT
ON Réservations
FOR EACH ROW
BEGIN
DECLARE reservation_count INT;

SELECT COUNT(*) INTO reservation_count
    FROM Réservations
WHERE voiture_id = NEW.voiture_id
      AND (
        (NEW.date_debut BETWEEN date_debut AND date_fin)
            OR
        (NEW.date_fin BETWEEN date_debut AND date_fin)
            OR
        (date_debut BETWEEN NEW.date_debut AND NEW.date_fin)
            OR
        (date_fin BETWEEN NEW.date_debut AND NEW.date_fin)
        );

IF reservation_count > 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'La voiture est déjà réservée pendant la période choisie. Il y a un chevauchement avec une réservation existante.';
    END IF;
END //



