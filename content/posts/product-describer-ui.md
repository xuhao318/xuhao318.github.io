---
title: "Building a ChatGPT-Style Product Description Assistant with AI Vision"
date: 2025-01-18
tags: ["react", "nodejs", "express", "ai", "llm", "qwen", "docker", "fullstack", "nedb"]
authors: ["Hao"]
---

# Building a ChatGPT-Style Product Description Assistant with AI Vision

In the era of e-commerce and digital marketing, creating compelling product descriptions is both an art and a science. What if we could harness the power of AI vision models to automatically generate high-quality product descriptions from images? In this post, I'll walk you through my journey building Product Describer UI - a full-stack application that combines a ChatGPT-style interface with AI-powered image analysis to create engaging product descriptions.

## The Problem

E-commerce businesses often struggle with:
- **Scale**: Hundreds or thousands of products need descriptions
- **Consistency**: Maintaining a uniform tone and quality across descriptions
- **Multilingual support**: Reaching global audiences requires translations
- **Time constraints**: Manual description writing is time-consuming
- **Visual analysis**: Extracting product features from images requires expertise

Traditional solutions involve hiring copywriters or using template-based systems, but these approaches are expensive and lack the flexibility to understand product images contextually.

## The Solution: AI-Powered Product Describer

Product Describer UI is a full-stack web application that leverages multimodal AI (specifically, the Qwen Vision-Language Model) to analyze product images and generate descriptions through natural conversation. The ChatGPT-style interface allows users to:

- Upload product images with each message
- Chat with AI to refine descriptions
- Maintain conversation history for iterative improvements
- Track all conversations and generated content

## Architecture Overview

The application follows a modern full-stack architecture:

```
┌─────────────────────────────────────────┐
│         Frontend (React + Vite)         │
│    - ChatGPT-style UI                   │
│    - Image upload per message           │
│    - Conversation history               │
└──────────────┬──────────────────────────┘
               │ HTTP/REST
               │
┌──────────────▼──────────────────────────┐
│       Backend (Node.js + Express)       │
│    - RESTful API                        │
│    - Session management                 │
│    - Authentication                     │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────┐
        │             │
┌───────▼──────┐  ┌──▼──────────────┐
│ NeDB Storage │  │  Qwen LLM API   │
│ - Chats      │  │  - Vision model │
│ - Products   │  │  - Text gen     │
│ - Tasks      │  │                 │
└──────────────┘  └─────────────────┘
```

### Technology Stack

**Frontend:**
- React 18 with functional components and hooks
- Vite for blazing-fast development and builds
- CSS for styling with ChatGPT-inspired design
- Axios for API communication

**Backend:**
- Node.js runtime
- Express.js web framework
- NeDB for lightweight embedded database
- express-session for authentication
- Multer for handling image uploads

**AI Integration:**
- Qwen Vision-Language Model API
- Multimodal capability (text + images)
- Base64 image encoding

**Deployment:**
- Docker and Docker Compose
- Multi-stage builds for optimization
- Nginx for serving frontend
- Environment-based configuration

## Key Implementation Details

### 1. ChatGPT-Style Interface

The frontend provides a familiar chat experience where users can interact with the AI naturally:

```javascript
// Simplified message component structure
const ChatMessage = ({ message, isUser }) => {
  return (
    <div className={`message ${isUser ? 'user' : 'assistant'}`}>
      {message.image && (
        <img src={message.image} alt="Product" />
      )}
      <div className="message-text">
        {message.content}
      </div>
    </div>
  );
};
```

Each message can contain:
- Text content (required)
- An optional product image
- Metadata (timestamp, conversation ID, etc.)

### 2. Image Handling Pipeline

One of the key challenges was implementing efficient image handling:

**Frontend Upload:**
```javascript
const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // Validate file type
  if (!file.type.startsWith('image/')) {
    alert('Please select an image file');
    return;
  }

  // Convert to base64 for API transmission
  const reader = new FileReader();
  reader.onload = (e) => {
    setImagePreview(e.target.result);
    setSelectedImage(e.target.result);
  };
  reader.readAsDataURL(file);
};
```

**Backend Processing:**
```javascript
// API endpoint for chat with image
app.post('/chat', async (req, res) => {
  const { prompt, image, conversationId } = req.body;

  try {
    // Call Qwen API with text and optional image
    const aiResponse = await qwenService.generateResponse({
      prompt,
      image,
      conversationId
    });

    // Store conversation in database
    const chatRecord = {
      conversationId: aiResponse.conversationId,
      userMessage: { prompt, image },
      aiMessage: aiResponse.message,
      timestamp: new Date()
    };

    await db.chats.insert(chatRecord);

    res.json(aiResponse);
  } catch (error) {
    console.error('Chat error:', error);
    res.status(500).json({ error: error.message });
  }
});
```

### 3. Qwen Vision-Language Model Integration

The Qwen service handles the AI interaction:

```javascript
class QwenService {
  constructor(apiKey, apiUrl) {
    this.apiKey = apiKey;
    this.apiUrl = apiUrl;
    this.conversationHistory = new Map();
  }

  async generateResponse({ prompt, image, conversationId }) {
    // Get or create conversation history
    const history = this.conversationHistory.get(conversationId) || [];

    // Build message with multimodal content
    const messages = [
      ...history,
      {
        role: 'user',
        content: [
          { type: 'text', text: prompt },
          ...(image ? [{
            type: 'image_url',
            image_url: { url: image }
          }] : [])
        ]
      }
    ];

    // Call Qwen API
    const response = await axios.post(
      this.apiUrl,
      {
        model: 'qwen-vl-plus',
        messages
      },
      {
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json'
        }
      }
    );

    const aiMessage = response.data.choices[0].message;

    // Update conversation history
    history.push(
      messages[messages.length - 1],
      aiMessage
    );
    this.conversationHistory.set(conversationId, history);

    return {
      message: aiMessage.content,
      conversationId
    };
  }
}
```

### 4. Conversation State Management

Managing conversation state across the stack was crucial:

**Frontend State:**
```javascript
const [conversations, setConversations] = useState([]);
const [currentConversationId, setCurrentConversationId] = useState(null);
const [messages, setMessages] = useState([]);

const sendMessage = async (text, image) => {
  // Optimistic UI update
  const userMessage = { text, image, isUser: true };
  setMessages(prev => [...prev, userMessage]);

  try {
    const response = await api.post('/chat', {
      prompt: text,
      image,
      conversationId: currentConversationId
    });

    // Add AI response to messages
    setMessages(prev => [...prev, {
      text: response.data.message,
      isUser: false
    }]);

    // Update conversation ID
    if (!currentConversationId) {
      setCurrentConversationId(response.data.conversationId);
    }
  } catch (error) {
    // Handle error and remove optimistic message
    setMessages(prev => prev.slice(0, -1));
    alert('Failed to send message: ' + error.message);
  }
};
```

**Backend Persistence with NeDB:**
```javascript
// Database initialization
const Datastore = require('nedb');

const db = {
  chats: new Datastore({ filename: './data/chats.db', autoload: true }),
  products: new Datastore({ filename: './data/products.db', autoload: true }),
  tasks: new Datastore({ filename: './data/tasks.db', autoload: true })
};

// Create indexes for efficient queries
db.chats.ensureIndex({ fieldName: 'conversationId' });
db.chats.ensureIndex({ fieldName: 'timestamp' });
```

### 5. Authentication and Security

The application implements session-based authentication:

```javascript
const session = require('express-session');

app.use(session({
  secret: process.env.SESSION_SECRET || 'dev-secret-change-in-prod',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));

// Authentication middleware
const requireAuth = (req, res, next) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
};

// Protected routes
app.post('/chat', requireAuth, async (req, res) => {
  // ... chat logic
});
```

## Docker Deployment Strategy

One of my goals was to make deployment as simple as possible. The project uses Docker Compose with separate containers for frontend and backend.

### Multi-Stage Frontend Build

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

ARG REACT_APP_BACKEND_URL
ENV REACT_APP_BACKEND_URL=$REACT_APP_BACKEND_URL

RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Runtime environment configuration
RUN echo "window.__REACT_APP_BACKEND_URL__='${REACT_APP_BACKEND_URL}';" \
    > /usr/share/nginx/html/env-config.js

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Backend Container

```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Create data directory for NeDB
RUN mkdir -p /usr/src/app/data

EXPOSE 5050
CMD ["node", "src/server.js"]
```

### Docker Compose Orchestration

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
    ports:
      - "5050:5050"
    environment:
      - NODE_ENV=production
      - FRONTEND_ORIGIN=${FRONTEND_ORIGIN:-http://localhost}
    env_file:
      - backend/.env
    volumes:
      - backend-data:/usr/src/app/data
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      args:
        REACT_APP_BACKEND_URL: /api
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  backend-data:
```

### Handling CORS and Proxying

A key challenge in deployment was handling cross-origin requests. The solution uses Nginx as a reverse proxy:

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    # Serve static files
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to backend
    location /api {
        proxy_pass http://backend:5050;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

This configuration:
- Serves React app from root
- Proxies `/api/*` requests to the backend
- Eliminates CORS issues
- Provides a single-origin experience

## Running in Virtual Machines

For testing in VM environments (VirtualBox, VMware, etc.), the application supports flexible networking:

### Bridged Network Mode
```bash
# VM gets its own IP on the LAN
# Access at: http://<vm-ip>

# Set in backend/.env:
FRONTEND_ORIGIN=http://192.168.1.100
```

### NAT with Port Forwarding
```bash
# VirtualBox example
VBoxManage modifyvm "ProductDescriber" --natpf1 "http,tcp,,8080,,80"
VBoxManage modifyvm "ProductDescriber" --natpf1 "api,tcp,,5050,,5050"

# Access at: http://localhost:8080
```

## Lessons Learned

### 1. Monorepo with npm Workspaces

Using npm workspaces simplified dependency management:

```json
{
  "name": "product-describer-ui",
  "private": true,
  "workspaces": [
    "frontend",
    "backend"
  ],
  "scripts": {
    "dev": "concurrently \"npm start -w frontend\" \"npm start -w backend\"",
    "test": "npm test --workspaces",
    "build": "npm run build --workspaces"
  }
}
```

Benefits:
- Single `npm install` for the entire project
- Shared dependencies are hoisted
- Easy to run scripts across packages
- Better for CI/CD pipelines

### 2. Base64 vs Multipart/Form-Data

Initially, I considered using multipart/form-data for image uploads, but base64 encoding proved simpler:

**Advantages:**
- Works seamlessly with JSON APIs
- Easier to store in NeDB
- Simpler client-side implementation
- Compatible with Qwen API format

**Disadvantages:**
- ~33% larger payload size
- Higher memory usage
- Not ideal for very large images

For production at scale, I would implement:
- Image compression before encoding
- Cloud storage (S3, etc.) with URL references
- CDN for image delivery

### 3. NeDB vs Traditional Databases

NeDB is perfect for this use case:

**Pros:**
- Zero configuration
- File-based (easy backups)
- MongoDB-like API
- Great for Docker volumes
- No external dependencies

**Cons:**
- Not suitable for high-concurrency
- Limited query optimization
- File I/O can be slow at scale

For production scaling, I'd migrate to:
- MongoDB for familiar API
- PostgreSQL with JSONB for relational needs
- Redis for session storage

### 4. Environment Configuration Complexity

Managing environment variables across development and production was tricky:

**Solution:**
```javascript
// Runtime config injection
window.__REACT_APP_BACKEND_URL__ = '/api';

// In React app
const API_URL = window.__REACT_APP_BACKEND_URL__ ||
                process.env.REACT_APP_BACKEND_URL ||
                'http://localhost:5050';
```

This allows the same Docker image to work in different environments.

### 5. Session Management in Containers

Initial session issues were resolved by:
- Using persistent volumes for NeDB
- Environment-based session secrets
- Proper cookie configuration for different origins

## Future Enhancements

Several improvements are on my roadmap:

### 1. Batch Processing
Allow users to upload multiple images for bulk description generation:
```javascript
POST /api/batch-describe
{
  images: [base64_1, base64_2, ...],
  template: "E-commerce product description"
}
```

### 2. Template System
Predefined templates for different industries:
- E-commerce (feature-focused)
- Real estate (location and amenities)
- Fashion (style and materials)
- Food & beverage (ingredients and taste)

### 3. Export Functionality
Export descriptions in various formats:
- CSV for bulk import
- JSON for API integration
- Markdown for content systems
- HTML for direct publishing

### 4. Multilingual Support
Leverage Qwen's multilingual capabilities:
- Language detection from images
- Multi-language output
- Translation of existing descriptions

### 5. Analytics Dashboard
Track usage metrics:
- Number of descriptions generated
- Most used conversation flows
- Image types and categories
- API usage and costs

### 6. WebSocket for Real-Time Updates
Replace polling with WebSocket for better UX:
```javascript
const ws = new WebSocket('ws://backend:5050');
ws.on('message', (data) => {
  // Stream AI response in real-time
  updateMessage(data.chunk);
});
```

## Performance Considerations

### Current Performance
- Frontend initial load: ~500ms (gzipped)
- API response time: 2-5s (depends on Qwen API)
- Image upload limit: 5MB
- Concurrent users: ~100 (NeDB limitation)

### Optimization Strategies

**1. Image Optimization:**
```javascript
// Client-side compression before upload
const compressImage = async (file, maxWidth = 1024) => {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  const img = await loadImage(file);
  const ratio = Math.min(maxWidth / img.width, 1);

  canvas.width = img.width * ratio;
  canvas.height = img.height * ratio;

  ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
  return canvas.toDataURL('image/jpeg', 0.8);
};
```

**2. Caching Strategy:**
```javascript
// Cache similar prompts + images
const cacheKey = crypto
  .createHash('md5')
  .update(prompt + imageHash)
  .digest('hex');

if (cache.has(cacheKey)) {
  return cache.get(cacheKey);
}
```

**3. Rate Limiting:**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/chat', limiter);
```

## Cost Analysis

Running this application in production has several cost components:

### Qwen API Costs
- Vision model: ~$0.01 per request
- Text generation: ~$0.001 per 1K tokens
- Monthly estimate (1000 users, 10 requests/user): ~$100

### Infrastructure Costs
- Small VPS (2GB RAM): $5-10/month
- Docker-friendly hosting (DigitalOcean): $12/month
- CDN for images (Cloudflare): Free tier sufficient
- Total: ~$20-25/month + API costs

### Scaling Costs
- Add Redis for sessions: +$5/month
- Upgrade to PostgreSQL: +$10/month
- Load balancer: +$10/month
- For 10K users: ~$50/month infrastructure + ~$1000 API costs

## Conclusion

Building Product Describer UI was a fascinating journey into combining modern web development with AI capabilities. The project demonstrates:

- **Full-stack integration**: Seamless communication between React frontend and Node.js backend
- **AI vision applications**: Practical use of multimodal language models
- **Docker deployment**: Production-ready containerization
- **User experience**: ChatGPT-style interface for natural interaction
- **Scalability considerations**: Architecture ready for growth

The combination of a familiar chat interface with AI vision capabilities makes product description generation accessible and efficient. While there's room for optimization and additional features, the current implementation provides a solid foundation for AI-powered content creation.

Whether you're building an e-commerce platform, a content management system, or exploring AI applications, the patterns and practices in this project can serve as a reference for integrating vision-language models into your applications.

The complete source code is available on [GitHub](https://github.com/hxu/product-describer-ui), and I welcome contributions and feedback from the community.

## Key Takeaways

1. **Start simple**: NeDB and monorepo structure kept initial development fast
2. **Docker early**: Containerization from day one saved deployment headaches
3. **User experience matters**: ChatGPT-style interface provides familiar, intuitive interaction
4. **Plan for scale**: Architecture allows gradual migration to more robust infrastructure
5. **AI APIs are powerful**: Qwen Vision-Language Model provides impressive capabilities with minimal integration effort

In future posts, I'll explore advanced topics like implementing real-time streaming responses, building a template system for different industries, and scaling the application to handle thousands of concurrent users.

## References

- [Qwen Vision-Language Model Documentation](https://qwen.readthedocs.io/)
- [React 18 Documentation](https://react.dev/)
- [Docker Compose Best Practices](https://docs.docker.com/compose/production/)
- [NeDB Documentation](https://github.com/louischatriot/nedb)
- [Express.js Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
