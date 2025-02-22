// Importing libraries
const express = require('express');
const mariadb = require('mariadb');
const bodyParser = require('body-parser');
const axios = require('axios');

const app = express();
const PORT = process.env.PORT || 8080;

app.use(bodyParser.json());

// MariaDB Connection Pool
const pool = mariadb.createPool({
    host: '127.0.0.1',         
    user: 'root',               
    password: 'hw2db@123A',     
    database: 'hw2db',
    connectionLimit: 20,       
    acquireTimeout: 15000      
});

const OTHER_VM_IP = process.env.OTHER_VM_IP; //to replicate to the other VM

// Confirming connection
(async () => {
    try {
        const conn = await pool.getConnection();
        console.log('Connected to MariaDB!');
        conn.release();
    } catch (err) {
        console.error('Error connecting to MariaDB:', err);
    }
})();

// greeting
app.get('/greeting', (req, res) => {
    res.send('<h1>Hello World!</h1>');
});

// register
app.post('/register', async (req, res) => {
    const { username } = req.body;
    if (!username) {
        return res.status(400).json({ error: 'Username is required' });
    }

    let conn;
    try {
        conn = await pool.getConnection();
        await conn.query('INSERT INTO Users (username) VALUES (?)', [username]);

        // Replicating to the other VM
        if (OTHER_VM_IP) {
            try {
                await axios.post(`http://${OTHER_VM_IP}:8080/register`, { username });
            } catch (err) {
                console.error(`Replication to ${OTHER_VM_IP} failed: ${err.message}`);
            }
        }
        res.status(201).json({ message: `User '${username}' registered successfully` });
    } catch (err) {
        if (err.code === 'ER_DUP_ENTRY') {
            return res.status(400).json({ error: 'Username already exists' });
        }
        res.status(500).json({ error: err.message });
    } finally {
        if (conn) conn.release();
    }
});

// list
app.get('/list', async (req, res) => {
    let conn;
    try {
        conn = await pool.getConnection();
        const rows = await conn.query('SELECT username FROM Users');
        const users = rows.map(row => row.username);
        res.json({ users });
    } catch (err) {
        res.status(500).json({ error: err.message });
    } finally {
        if (conn) conn.release(); 
    }
});

// clear
app.post('/clear', async (req, res) => {
    let conn;
    try {
        conn = await pool.getConnection();
        await conn.query('DELETE FROM Users');

        // Replicate clear operation to the other VM if IP is set
        if (OTHER_VM_IP) {
            try {
                await axios.post(`http://${OTHER_VM_IP}:8080/clear`);
            } catch (err) {
                console.error(`Clear operation on ${OTHER_VM_IP} failed: ${err.message}`);
            }
        }

        res.json({ message: 'All users have been removed' });
    } catch (err) {
        res.status(500).json({ error: err.message });
    } finally {
        if (conn) conn.release();
    }
});

// Server start
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server is running on http://0.0.0.0:${PORT}`);
});
