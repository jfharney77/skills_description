# Influencer Tracking Web Application Development Plan

This plan outlines the development of a FastAPI + React web application that allows users to select from three influencers (Obama, Musk, Cuban) and view their news summaries and Twitter feeds with sentiment analysis badges.

## Project Structure

```
influencer-tracking/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py (FastAPI app)
│   │   ├── models.py (SQLAlchemy models)
│   │   ├── database.py (SQLite setup)
│   │   ├── schemas.py (Pydantic schemas)
│   │   ├── crud.py (Database operations)
│   │   ├── mock_data.py (Mock data generation)
│   │   ├── sentiment.py (Sentiment analysis)
│   │   └── routers/
│   │       ├── influencers.py
│   │       ├── news.py
│   │       └── tweets.py
│   ├── requirements.txt
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── InfluencerDropdown.jsx
│   │   │   ├── NewsPanel.jsx
│   │   │   ├── TwitterFeed.jsx
│   │   │   └── SentimentBadge.jsx
│   │   ├── services/
│   │   │   └── api.js
│   │   └── index.css
│   ├── package.json
│   └── tailwind.config.js
└── README.md
```

## Backend Implementation (FastAPI)

### Database Schema (SQLite)
- **influencers**: id, name, handle, description
- **news**: id, influencer_id, title, summary, url, published_at
- **tweets**: id, influencer_id, content, created_at, sentiment_score, sentiment_label

### Key Dependencies
- fastapi, uvicorn
- sqlalchemy, sqlite3
- pydantic
- textblob (sentiment analysis)
- python-dotenv

### API Endpoints
- `GET /api/influencers` - List all influencers
- `GET /api/influencers/{id}/news` - Get news for specific influencer
- `GET /api/influencers/{id}/tweets` - Get tweets for specific influencer
- `POST /api/refresh-data` - Refresh mock data (for testing)

### Mock Data Generation
- Generate 10-15 news articles per influencer with realistic headlines
- Generate 20-30 tweets per influencer with varied content
- Apply sentiment analysis during data generation

### Sentiment Analysis
- Use TextBlob library for sentiment scoring
- Categorize as: positive (score > 0.1), neutral (-0.1 to 0.1), negative (score < -0.1)
- Store sentiment_score and sentiment_label in database

## Frontend Implementation (React + Tailwind)

### Key Dependencies
- react, react-dom
- axios (API calls)
- tailwindcss
- react-router-dom (optional, for future expansion)

### Components
1. **InfluencerDropdown**: Dropdown menu to select from 3 influencers
2. **NewsPanel**: Display news cards with title, summary, and link
3. **TwitterFeed**: Display 5 tweet cards with content and sentiment badges
4. **SentimentBadge**: Color-coded badge (green=positive, gray=neutral, red=negative)

### UI Layout
- Header with app title
- Main content area with:
  - Top: Influencer dropdown
  - Middle: Two-column layout (News panel | Twitter feed)
  - Each panel shows data for selected influencer
- Responsive design for mobile (stack panels vertically)

## Implementation Steps

### Phase 1: Backend Setup
1. Initialize FastAPI project structure
2. Set up SQLite database with SQLAlchemy
3. Create database models (influencers, news, tweets)
4. Implement mock data generation script
5. Add sentiment analysis using TextBlob
6. Create API endpoints for influencers, news, tweets
7. Seed database with initial mock data
8. Test all API endpoints

### Phase 2: Frontend Setup
1. Initialize React project with Vite
2. Install and configure Tailwind CSS
3. Set up axios for API communication
4. Create base layout with header
5. Implement InfluencerDropdown component
6. Implement NewsPanel component
7. Implement TwitterFeed component
8. Implement SentimentBadge component
9. Connect components to API endpoints
10. Add responsive styling

### Phase 3: Integration & Testing
1. Connect frontend to backend APIs
2. Test full user flow (select influencer → view data)
3. Verify sentiment badges display correctly
4. Test responsive design on different screen sizes
5. Add error handling for API failures
6. Polish UI/UX with loading states

### Phase 4: Railway Deployment (Future)
1. Create Railway project
2. Set up environment variables
3. Configure build process for both frontend and backend
4. Deploy and test production build

## Technical Decisions

- **SQLite**: Simple, file-based database perfect for this use case
- **TextBlob**: Lightweight Python sentiment analysis library
- **Tailwind CSS**: Utility-first CSS for rapid UI development
- **Vite**: Fast React build tool for development
- **Axios**: Promise-based HTTP client for API calls

## Notes

- No authentication required (public-facing)
- Data refreshed via API endpoint (not automated)
- Sentiment analysis performed during data generation
- Parallel display of news and tweets as shown in diagram
- 5 tweets displayed per influencer (as per diagram specification)
