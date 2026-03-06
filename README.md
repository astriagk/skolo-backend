# Skolo Backend

A backend server for the Skolo platform, built with Node.js, Express, MongoDB, and WebSocket support — providing both RESTful APIs and real-time Socket APIs.

## Tech Stack

- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: MongoDB
- **Real-time**: WebSocket (Socket.io)
- **API Types**: RESTful APIs, Real-time Socket APIs

## Features

- RESTful API endpoints for standard CRUD operations
- Real-time communication via WebSocket / Socket APIs
- MongoDB as the primary database
- Express.js for routing and middleware management

## Getting Started

### Prerequisites

- Node.js (v18+)
- MongoDB

### Installation

```bash
# Clone the repository
git clone <repository-url>
cd skolo-backend

# Install dependencies
npm install

# Start the server
npm start
```

### Environment Variables

Create a `.env` file in the root directory and configure the following:

```env
PORT=3000
MONGO_URI=mongodb://localhost:27017/skolo
```
