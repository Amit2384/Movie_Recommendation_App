# Movie Recommendation App - React & Appwrite

A modern movie discovery application built with React, Vite, and Appwrite that allows users to search for movies and track trending searches.

## 🚀 Features

- **Movie Search**: Search for movies using The Movie Database (TMDB) API
- **Trending Movies**: Track and display most searched movies
- **Debounced Search**: Optimized search with 500ms delay to reduce API calls
- **Responsive Design**: Modern UI with loading states and error handling
- **Real-time Data**: Appwrite backend for search analytics

## 📁 Project Structure

```
src/
├── components/
│   ├── MovieCard.jsx     # Individual movie display component
│   ├── search.jsx        # Search input component
│   └── spinner.jsx       # Loading spinner component
├── App.jsx               # Main application component
├── appwrite.js           # Appwrite backend integration
└── main.jsx              # React app entry point
```

## 🔧 Core Components Documentation

### 1. App.jsx - Main Application Component

**Purpose**: The root component that orchestrates the entire application

#### Key Imports & Setup
```javascript
import {useDebounce, useRafState} from 'react-use'
import {updateSearchCount, getTrendingMovies} from './appwrite.js'
```
- **useDebounce**: Prevents excessive API calls by waiting 500ms after user stops typing
- **Appwrite functions**: Handle search analytics and trending data

#### State Management
```javascript
const [searchTerm, setSearchTerm] = useState('');           // Current search input
const [errorMessage, setErrorMessage] = useState('');       // Error display
const [movieList, setMovieList] = useState([]);            // Search results
const [trendingMovies, setTrendingMovies] = useState([]);   // Popular searches
const [isLoading, setIsLoading] = useState(false);         // Loading state
const [debouncedSearchTerm, setDebouncedSearchTerm] = useState(''); // Optimized search
```

#### API Configuration
```javascript
const API_BASE_URL="https://api.themoviedb.org/3"
const API_KEY= import.meta.env.VITE_TMDB_API_KEY;
const API_OPTIONS = {
  method: 'GET',
  headers: {
    accept: 'application/json',
    Authorization: `Bearer ${API_KEY}`
  }
}
```
- **Why**: Centralizes TMDB API configuration with secure environment variable for API key

#### Debounced Search Implementation
```javascript
useDebounce(()=>setDebouncedSearchTerm(searchTerm),500,[searchTerm]);
```
- **Why**: Prevents API spam by waiting 500ms after user stops typing
- **How**: Only triggers search when user pauses, improving performance and reducing costs

#### fetchMovies Function
```javascript
const fetchMovies = async (query = '') => {
  const endpoint = query
    ? `${API_BASE_URL}/search/movie?query=${encodeURIComponent(query)}`
    : `${API_BASE_URL}/discover/movie?sort_by=popularity.desc`;
}
```
- **Purpose**: Handles both search and discovery of movies
- **Logic**: Uses search endpoint when query exists, otherwise shows popular movies
- **Analytics**: Calls `updateSearchCount()` to track successful searches

#### loadTrendingMovies Function
```javascript
const loadTrendingMovies = async () => {
  const movies = await getTrendingMovies();
  setTrendingMovies(movies);
}
```
- **Purpose**: Fetches most searched movies from Appwrite database
- **Why**: Shows users what others are searching for, improving discovery

### 2. MovieCard.jsx - Movie Display Component

**Purpose**: Renders individual movie information in a card format

#### Props Destructuring
```javascript
const MovieCard = ({movie : 
    {title, vote_average, poster_path, release_date, original_language}}) => {
```
- **Why**: Uses destructuring for cleaner code and better performance
- **Pattern**: Extracts only needed properties from movie object

#### Image Handling
```javascript
<img src={poster_path ? `https://image.tmdb.org/t/p/w500/${poster_path}` : '/no-movie.png'}
     alt={title} />
```
- **Purpose**: Displays movie poster with fallback
- **TMDB Image API**: Uses w500 size for optimal loading/quality balance
- **Fallback**: Shows placeholder when poster unavailable

#### Rating Display
```javascript
<p>{vote_average ? vote_average.toFixed(1) : 'N/A'}</p>
```
- **Why**: Formats rating to 1 decimal place for consistency
- **Fallback**: Shows 'N/A' when rating unavailable

#### Date Processing
```javascript
<p className='year'>
    {release_date ? release_date.split('-')[0] : 'N/A'}
</p>
```
- **Purpose**: Extracts year from full date string (YYYY-MM-DD → YYYY)
- **Why**: Year is more relevant for movie cards than full date

### 3. appwrite.js - Backend Integration

**Purpose**: Handles all backend operations for search analytics

#### Client Configuration
```javascript
const client = new Client()
.setEndpoint("https://nyc.cloud.appwrite.io/v1")
.setProject(PROJECT_ID)
```
- **Why**: Initializes Appwrite client with NYC endpoint for better performance
- **Security**: Uses environment variables for sensitive IDs

#### updateSearchCount Function
```javascript
export const updateSearchCount = async (searchTerm, movie) => {
  // Check if search term exists
  const result = await database.listDocuments(DATABASE_ID, COLLECTION_ID,
       [Query.equal('searchTerm', searchTerm)]
  );
  
  if(result.documents.length > 0){
    // Update existing count
    await database.updateDocument(DATABASE_ID, COLLECTION_ID, doc.$id, {
        count: doc.count + 1,
    });
  }else{
    // Create new search record
    await database.createDocument(DATABASE_ID, COLLECTION_ID, ID.unique(), {
        searchTerm,
        count: 1,
        movie_id: movie.id,
        poster_url : `https://image.tmdb.org/t/p/w500${movie.poster_path}`,
    });
  }
}
```
- **Logic**: Implements upsert pattern (update if exists, create if not)
- **Why**: Tracks search popularity for trending feature
- **Data**: Stores search term, count, movie ID, and poster URL

#### getTrendingMovies Function
```javascript
export const getTrendingMovies = async () => {
    const result = await database.listDocuments(DATABASE_ID, COLLECTION_ID, [
        Query.limit(5),
        Query.orderDesc('count'),
    ])
    return result.documents;
}
```
- **Purpose**: Retrieves top 5 most searched movies
- **Sorting**: Orders by count in descending order
- **Limit**: Returns only top 5 for performance and UI clarity

## 🔄 Data Flow

1. **User types** → Search input updates `searchTerm`
2. **Debounce hook** → Waits 500ms, then updates `debouncedSearchTerm`
3. **useEffect** → Triggers `fetchMovies()` when debounced term changes
4. **API Call** → Fetches movies from TMDB
5. **Analytics** → Updates search count in Appwrite (if successful search)
6. **UI Update** → Displays movies using `MovieCard` components
7. **Trending** → Loads trending movies on app initialization

## 🎯 Key Design Decisions

### Why Debouncing?
- **Problem**: Every keystroke would trigger an API call
- **Solution**: Wait 500ms after user stops typing
- **Benefit**: Reduces API calls by ~90%, improves performance

### Why Appwrite?
- **Real-time**: Tracks search analytics in real-time
- **Scalable**: Handles concurrent users efficiently
- **Simple**: Easy integration with React

### Why Component Separation?
- **Reusability**: MovieCard can be used anywhere
- **Maintainability**: Each component has single responsibility
- **Testing**: Easier to test individual components

## 🛠️ Environment Variables Required

```env
VITE_TMDB_API_KEY=your_tmdb_api_key
VITE_APPWRITE_PROJECT_ID=your_project_id
VITE_APPWRITE_DATABASE_ID=your_database_id
VITE_APPWRITE_COLLECTION_ID=your_collection_id
```

## 🚀 Getting Started

1. **Install dependencies**: `npm install`
2. **Set up environment variables** in `.env` file
3. **Run development server**: `npm run dev`
4. **Build for production**: `npm run build`

## 🔮 Future Enhancements

- Movie details page
- User favorites
- Advanced filtering
- Movie recommendations
- User authentication
