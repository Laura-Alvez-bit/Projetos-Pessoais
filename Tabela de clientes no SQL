CREATE TABLE clientes_laura (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    cep INT NOT NULL,
    numero INT NOT NULL,
    complemento VARCHAR(20),
    data_cadastro DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cep) REFERENCES cep_laura(cep)
)



//* EXERCÍCIOS SOBRE A TABELA DE CLIENTES *//



1) Exibir todos os clientes com nome completo que comece com a letra 'A':

SELECT *
FROM clientes
WHERE nome LIKE 'A%';

2a) Exibir todos os clientes com nome completo que termine com a letra 'S'

SELECT *
FROM clientes
WHERE nome LIKE '%S';

2b) Exibir todos os clientes com nome completo que contenha a sequência 'AN' em qualquer parte do nome

SELECT *
FROM clientes
WHERE nome LIKE '%AN%'

3) Exibir todos os clientes com sobrenome que comecem com a letra 'S':

SELECT *
FROM clientes
WHERE nome LIKE '% S%'

4) Exibir todos os clientes que tenham pelo menos 2 letras A em qualquer parte do nome completo:

SELECT *
FROM clientes
WHERE nome LIKE '%A%A%'

5) Exibir todos os clientes que tenham pelo menos 1 letra D no nome completo e termine com a letra S:

SELECT *
FROM clientes
WHERE nome LIKE '%D%S'

6) Exibir o nome e o email de apenas 5 clientes que comecem com a letra A:

SELECT nome, email
FROM clientes
WHERE nome LIKE 'A%' 
LIMIT 5

SOLUÇÃO PARA NÃO ANULAR A QUESTÃO:
SELECT nome, email
FROM clientes
WHERE nome LIKE 'A%' AND email LIKE 'A%'
LIMIT 5

7) Exibir o código, nome e a obs apenas de clientes que tenham alguma observação cadastrada:

SELECT id_cliente, nome, obs
FROM clientes
WHERE obs IS NOT NULL

8) Exibir o código, nome e email apenas dos clientes que moram em SP:

SELECT id_cliente, nome, email, estado
FROM clientes
INNER JOIN ceps ON clientes.cep = ceps.cep
WHERE estado = 'SP'

9) Exibir o código, nome e endereço completo dos clientes que moram em Apartamento, em ordem de estado e cidade:

SELECT id_cliente, nome, endereco, numero,
       complemento, bairro, cidade, estado, ceps.cep
FROM clientes
INNER JOIN ceps ON clientes.cep = ceps.cep
WHERE complemento LIKE 'AP%'
ORDER BY estado, cidade

10) Exibir o código, nome e endereço completo dos clientes que não moram no centro:

SELECT id_cliente, nome, endereco, numero,
       complemento, bairro, cidade, estado, ceps.cep
FROM clientes
INNER JOIN ceps ON clientes.cep = ceps.cep
WHERE bairro <> 'Centro'

SELECT id_cliente, nome, endereco, numero,
       complemento, bairro, cidade, estado, ceps.cep
FROM clientes
INNER JOIN ceps ON clientes.cep = ceps.cep
WHERE bairro NOT LIKE 'Centro%'
