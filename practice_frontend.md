# Простейшее React-приложение для книжного магазина


---

## Часть 1. Общая архитектура

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   React     │────▶│  Node.js    │────▶│ PostgreSQL  │
│  (Frontend) │     │  (Backend)  │     │   (БД)      │
└─────────────┘     └─────────────┘     └─────────────┘
     :3000              :5000              :5432
```

**Что вам понадобится:**
- Node.js (18+)
- PostgreSQL (уже есть)
- npm или yarn

---

## Часть 2. Настройка бэкенда (Node.js + Express)

### 2.1. Создайте папку проекта

```bash
mkdir bookstore-app
cd bookstore-app
mkdir backend
cd backend
```

### 2.2. Инициализируйте Node.js проект

```bash
npm init -y
```

### 2.3. Установите зависимости

```bash
npm install express pg cors
npm install -D nodemon
```

### 2.4. Создайте файл `server.js`

```javascript
const express = require('express');
const { Pool } = require('pg');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

// Подключение к PostgreSQL
const pool = new Pool({
    user: 'myuser',        // Ваш пользователь
    host: 'localhost',
    database: 'bookstore', // Имя вашей БД
    password: 'mypassword123', // Ваш пароль
    port: 5432,
});

// ===== API ЭНДПОИНТЫ =====

// 1. Получить все книги с авторами
app.get('/api/books', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT b.*, 
                   STRING_AGG(a.first_name || ' ' || a.last_name, ', ') as authors
            FROM books b
            LEFT JOIN book_authors ba ON b.id = ba.book_id
            LEFT JOIN authors a ON a.id = ba.author_id
            GROUP BY b.id
            ORDER BY b.id
        `);
        res.json(result.rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 2. Лучшая книга по продажам
app.get('/api/best-seller', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT b.title, SUM(s.quantity) as total_sold
            FROM sales s
            JOIN books b ON b.id = s.book_id
            GROUP BY b.id
            ORDER BY total_sold DESC
            LIMIT 1
        `);
        res.json(result.rows[0] || {});
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 3. Выручка по книгам
app.get('/api/revenue', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT b.title, SUM(s.price_at_sale * s.quantity) as revenue
            FROM sales s
            JOIN books b ON b.id = s.book_id
            GROUP BY b.id
            ORDER BY revenue DESC
        `);
        res.json(result.rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 4. Лучший покупатель
app.get('/api/best-customer', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT c.name, SUM(s.price_at_sale * s.quantity) as total_spent
            FROM sales s
            JOIN customers c ON c.id = s.customer_id
            GROUP BY c.id
            ORDER BY total_spent DESC
            LIMIT 1
        `);
        res.json(result.rows[0] || {});
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 5. Покупатели, купившие конкретную книгу
app.get('/api/customers-by-book/:bookTitle', async (req, res) => {
    const { bookTitle } = req.params;
    try {
        const result = await pool.query(`
            SELECT DISTINCT c.name, c.email
            FROM sales s
            JOIN books b ON b.id = s.book_id
            JOIN customers c ON c.id = s.customer_id
            WHERE b.title = $1
        `, [bookTitle]);
        res.json(result.rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 6. Остатки на складе
app.get('/api/stock', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT title, stock FROM books WHERE stock > 0 ORDER BY title
        `);
        res.json(result.rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 7. Популярный автор
app.get('/api/popular-author', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT a.first_name, a.last_name, SUM(s.quantity) as sold
            FROM authors a
            JOIN book_authors ba ON a.id = ba.author_id
            JOIN books b ON b.id = ba.book_id
            JOIN sales s ON s.book_id = b.id
            GROUP BY a.id
            ORDER BY sold DESC
            LIMIT 1
        `);
        res.json(result.rows[0] || {});
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// 8. Книги без продаж
app.get('/api/unsold-books', async (req, res) => {
    try {
        const result = await pool.query(`
            SELECT b.title
            FROM books b
            LEFT JOIN sales s ON b.id = s.book_id
            WHERE s.id IS NULL
        `);
        res.json(result.rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});

// Запуск сервера
const PORT = 5000;
app.listen(PORT, () => {
    console.log(`✅ Сервер запущен на http://localhost:${PORT}`);
});
```

### 2.5. Запустите бэкенд

```bash
node server.js
# Или с автоматическим перезапуском:
npx nodemon server.js
```

**Ожидаемый результат:**
```
✅ Сервер запущен на http://localhost:5000
```

---

## Часть 3. Создание React-фронтенда

### 3.1. Вернитесь в корневую папку и создайте React-приложение

```bash
cd ../  # вернулись в bookstore-app
npx create-react-app frontend
cd frontend
```

### 3.2. Установите дополнительные зависимости

```bash
npm install axios
```

### 3.3. Замените содержимое `src/App.js`

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

const API_URL = 'http://localhost:5000/api';

function App() {
    const [books, setBooks] = useState([]);
    const [revenue, setRevenue] = useState([]);
    const [bestSeller, setBestSeller] = useState(null);
    const [bestCustomer, setBestCustomer] = useState(null);
    const [popularAuthor, setPopularAuthor] = useState(null);
    const [stock, setStock] = useState([]);
    const [unsoldBooks, setUnsoldBooks] = useState([]);
    const [selectedBook, setSelectedBook] = useState('');
    const [customersForBook, setCustomersForBook] = useState([]);
    const [activeTab, setActiveTab] = useState('books');
    const [loading, setLoading] = useState(false);

    // Загрузка всех данных
    const loadAllData = async () => {
        setLoading(true);
        try {
            const [
                booksRes,
                revenueRes,
                bestSellerRes,
                bestCustomerRes,
                popularAuthorRes,
                stockRes,
                unsoldBooksRes,
            ] = await Promise.all([
                axios.get(`${API_URL}/books`),
                axios.get(`${API_URL}/revenue`),
                axios.get(`${API_URL}/best-seller`),
                axios.get(`${API_URL}/best-customer`),
                axios.get(`${API_URL}/popular-author`),
                axios.get(`${API_URL}/stock`),
                axios.get(`${API_URL}/unsold-books`),
            ]);

            setBooks(booksRes.data);
            setRevenue(revenueRes.data);
            setBestSeller(bestSellerRes.data);
            setBestCustomer(bestCustomerRes.data);
            setPopularAuthor(popularAuthorRes.data);
            setStock(stockRes.data);
            setUnsoldBooks(unsoldBooksRes.data);
        } catch (error) {
            console.error('Ошибка загрузки:', error);
        } finally {
            setLoading(false);
        }
    };

    // Поиск покупателей по книге
    const searchCustomersByBook = async () => {
        if (!selectedBook) return;
        try {
            const res = await axios.get(`${API_URL}/customers-by-book/${selectedBook}`);
            setCustomersForBook(res.data);
        } catch (error) {
            console.error('Ошибка:', error);
        }
    };

    useEffect(() => {
        loadAllData();
    }, []);

    return (
        <div className="App">
            <h1>📚 Книжный магазин</h1>
            
            {loading && <div className="loader">Загрузка...</div>}

            {/* Вкладки */}
            <div className="tabs">
                <button className={activeTab === 'books' ? 'active' : ''} onClick={() => setActiveTab('books')}>
                    📖 Все книги
                </button>
                <button className={activeTab === 'sales' ? 'active' : ''} onClick={() => setActiveTab('sales')}>
                    💰 Продажи
                </button>
                <button className={activeTab === 'analytics' ? 'active' : ''} onClick={() => setActiveTab('analytics')}>
                    📊 Аналитика
                </button>
                <button className={activeTab === 'search' ? 'active' : ''} onClick={() => setActiveTab('search')}>
                    🔍 Поиск
                </button>
            </div>

            {/* Вкладка: Все книги */}
            {activeTab === 'books' && (
                <div className="tab-content">
                    <h2>Все книги</h2>
                    <table>
                        <thead>
                            <tr><th>ID</th><th>Название</th><th>Цена</th><th>Остаток</th><th>Авторы</th></tr>
                        </thead>
                        <tbody>
                            {books.map(book => (
                                <tr key={book.id}>
                                    <td>{book.id}</td>
                                    <td>{book.title}</td>
                                    <td>{book.price} ₽</td>
                                    <td>{book.stock}</td>
                                    <td>{book.authors || '—'}</td>
                                </tr>
                            ))}
                        </tbody>
                    </table>

                    <h3>📦 Остатки на складе</h3>
                    <ul>
                        {stock.map(item => (
                            <li key={item.title}>{item.title}: {item.stock} шт.</li>
                        ))}
                    </ul>

                    <h3>📭 Книги без продаж</h3>
                    <ul>
                        {unsoldBooks.map(book => (
                            <li key={book.title}>{book.title}</li>
                        ))}
                    </ul>
                </div>
            )}

            {/* Вкладка: Продажи */}
            {activeTab === 'sales' && (
                <div className="tab-content">
                    <h2>Выручка по книгам</h2>
                    <table>
                        <thead><tr><th>Книга</th><th>Выручка</th></tr></thead>
                        <tbody>
                            {revenue.map(item => (
                                <tr key={item.title}>
                                    <td>{item.title}</td>
                                    <td>{Number(item.revenue).toFixed(2)} ₽</td>
                                </tr>
                            ))}
                        </tbody>
                    </table>

                    <div className="stats">
                        <div className="stat-card">
                            <h3>🏆 Лучшая книга</h3>
                            <p>{bestSeller?.title || '—'}</p>
                            <small>Продано: {bestSeller?.total_sold || 0} шт.</small>
                        </div>
                        <div className="stat-card">
                            <h3>⭐ Лучший покупатель</h3>
                            <p>{bestCustomer?.name || '—'}</p>
                            <small>Потратил: {Number(bestCustomer?.total_spent || 0).toFixed(2)} ₽</small>
                        </div>
                        <div className="stat-card">
                            <h3>✍️ Популярный автор</h3>
                            <p>{popularAuthor?.first_name} {popularAuthor?.last_name}</p>
                            <small>Продано книг: {popularAuthor?.sold || 0}</small>
                        </div>
                    </div>
                </div>
            )}

            {/* Вкладка: Аналитика */}
            {activeTab === 'analytics' && (
                <div className="tab-content">
                    <h2>Группировка и агрегация</h2>
                    
                    <h3>Книги по авторам</h3>
                    <table>
                        <thead><tr><th>Автор</th><th>Книги</th></tr></thead>
                        <tbody>
                            {books
                                .filter(b => b.authors)
                                .reduce((acc, book) => {
                                    const authors = book.authors.split(', ');
                                    authors.forEach(author => {
                                        const existing = acc.find(item => item.author === author);
                                        if (existing) {
                                            existing.books.push(book.title);
                                        } else {
                                            acc.push({ author, books: [book.title] });
                                        }
                                    });
                                    return acc;
                                }, [])
                                .map(item => (
                                    <tr key={item.author}>
                                        <td>{item.author}</td>
                                        <td>{item.books.join(', ')}</td>
                                    </tr>
                                ))
                            }
                        </tbody>
                    </table>
                </div>
            )}

            {/* Вкладка: Поиск */}
            {activeTab === 'search' && (
                <div className="tab-content">
                    <h2>Кто купил книгу?</h2>
                    <div className="search-box">
                        <select value={selectedBook} onChange={(e) => setSelectedBook(e.target.value)}>
                            <option value="">Выберите книгу</option>
                            {books.map(book => (
                                <option key={book.id} value={book.title}>{book.title}</option>
                            ))}
                        </select>
                        <button onClick={searchCustomersByBook}>Найти</button>
                    </div>
                    
                    {customersForBook.length > 0 && (
                        <div>
                            <h3>Покупатели:</h3>
                            <ul>
                                {customersForBook.map(customer => (
                                    <li key={customer.email}>{customer.name} ({customer.email})</li>
                                ))}
                            </ul>
                        </div>
                    )}
                </div>
            )}
        </div>
    );
}

export default App;
```

### 3.4. Добавьте стили в `src/App.css`

```css
.App {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    font-family: Arial, sans-serif;
}

h1 {
    text-align: center;
    color: #333;
}

.tabs {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
    border-bottom: 1px solid #ddd;
    padding-bottom: 10px;
}

.tabs button {
    padding: 10px 20px;
    background: #f0f0f0;
    border: none;
    cursor: pointer;
    font-size: 16px;
    border-radius: 5px;
    transition: background 0.3s;
}

.tabs button:hover {
    background: #e0e0e0;
}

.tabs button.active {
    background: #007bff;
    color: white;
}

.tab-content {
    padding: 20px;
    background: #f9f9f9;
    border-radius: 10px;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin: 15px 0;
}

th, td {
    padding: 10px;
    text-align: left;
    border-bottom: 1px solid #ddd;
}

th {
    background: #007bff;
    color: white;
}

tr:hover {
    background: #f1f1f1;
}

.stats {
    display: flex;
    gap: 20px;
    margin: 20px 0;
    flex-wrap: wrap;
}

.stat-card {
    flex: 1;
    min-width: 200px;
    background: white;
    padding: 15px;
    border-radius: 10px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    text-align: center;
}

.stat-card h3 {
    margin: 0 0 10px 0;
    color: #007bff;
}

.stat-card p {
    font-size: 20px;
    font-weight: bold;
    margin: 10px 0;
}

.loader {
    text-align: center;
    padding: 20px;
    color: #007bff;
}

.search-box {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
}

.search-box select {
    flex: 1;
    padding: 10px;
    font-size: 16px;
}

.search-box button {
    padding: 10px 20px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.search-box button:hover {
    background: #0056b3;
}

ul {
    list-style: none;
    padding: 0;
}

li {
    padding: 8px;
    border-bottom: 1px solid #eee;
}
```

### 3.5. Запустите React-приложение

```bash
npm start
```

Откройте браузер по адресу: `http://localhost:3000`

---

## Часть 4. Проверка работы

После запуска вы увидите:

1. **Вкладка "Все книги"** — список книг с авторами, остатки, книги без продаж
2. **Вкладка "Продажи"** — выручка по книгам, лучшая книга, лучший покупатель, популярный автор
3. **Вкладка "Аналитика"** — группировка книг по авторам
4. **Вкладка "Поиск"** — поиск покупателей по названию книги

---

## Часть 5. Структура проекта (итоговая)

```
bookstore-app/
├── backend/
│   ├── package.json
│   ├── server.js
│   └── node_modules/
└── frontend/
    ├── package.json
    ├── public/
    └── src/
        ├── App.js
        ├── App.css
        └── index.js
```

---

## Часть 6. Запуск всего приложения (быстрая инструкция)

```bash
# Терминал 1: Запуск бэкенда
cd bookstore-app/backend
node server.js

# Терминал 2: Запуск фронтенда
cd bookstore-app/frontend
npm start
```
