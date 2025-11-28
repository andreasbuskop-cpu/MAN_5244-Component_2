l planning information

### 6. Issue Reporting & Impact Analysis

**Reporting System:**
- Captains report operational issues (mechanical, weather, navigation, cargo, crew)
- AI-powered impact analysis using Gemini
- Severity classification (low, medium, high, critical)

**Impact Analysis:**
- Estimated delay calculations
- Financial cost assessment
- Charter party penalty estimations
- Recommended mitigation actions
- Risk level determination

**Issue Types:**
- Mechanical failures
- Weather-related delays
- Navigation issues
- Cargo problems
- Crew situations

## Database Schema

### Tables

**Users**
- id (Primary Key)
- username (unique)
- email
- password_hash (Werkzeug security)
- full_name
- role (operator, captain, port_agent)
- created_at

**Vessels**
- id (Primary Key)
- mmsi (unique, 9-digit identifier)
- imo (International Maritime Organization number)
- name
- ship_type / ship_type_code (70-79 for cargo)
- flag_country
- length_m / width_m
- built_year
- owner_company
- operator_id (Foreign Key → Users)
- fuel_consumption_tons_per_day
- fleet_assignment (my_fleet, competing_fleet, unassigned)
- latitude / longitude (current position)
- speed_knots / heading_degrees
- destination / eta
- captain_name / captain_contact
- port_agent_name / port_agent_contact
- planned_arrival_port / planned_arrival_time
- last_position_update
- is_active (soft-delete flag)
- deactivated_at

**VesselPositions**
- id (Primary Key)
- vessel_id (Foreign Key → Vessels)
- latitude / longitude
- speed_knots / heading_degrees
- recorded_at

**Routes**
- id (Primary Key)
- vessel_id (Foreign Key → Vessels)
- operator_id (Foreign Key → Users)
- origin_port / destination_port
- origin_lat / origin_lon
- destination_lat / destination_lon
- distance_nm (nautical miles)
- current_speed_knots / recommended_speed_knots
- estimated_fuel_consumption
- estimated_cost_usd
- optimized_fuel_consumption
- optimized_cost_usd
- savings_usd / savings_percentage
- estimated_arrival
- ai_rationale
- weather_risks / operational_risks
- created_at

**Issues**
- id (Primary Key)
- vessel_id (Foreign Key → Vessels)
- captain_id (Foreign Key → Users)
- issue_type (mechanical, weather, navigation, cargo, crew)
- description
- severity (low, medium, high, critical)
- status (reported, investigating, resolved, escalated)
- impact_analysis_json (AI analysis results)
- reported_at / resolved_at

### Database Optimizations

**Indexes:**
- ship_type_code (vessel filtering)
- fleet_assignment (fleet queries)
- mmsi (vessel lookups)
- is_active (active vessel filtering)

**Concurrency Handling:**
- SQLite WAL (Write-Ahead Logging) mode
- 30-second busy timeout
- Connection pooling with pre-ping
- Pool recycle every 300 seconds

**Data Management:**
- Soft-delete system for vessels (is_active flag)
- Automatic deactivation after 2 hours without AIS updates
- Preserves historical data for issue tracking
- Schema migration system for updates

## API Endpoints

### Authentication
- `POST /auth/login` - User authentication with session creation
- `GET /auth/logout` - Session termination

### Vessels
- `GET /api/vessels` - List all active vessels with fresh AIS data
- `GET /api/vessels/<id>` - Get detailed vessel information

### Route Optimization
- `POST /api/routes/optimize` - Generate AI-optimized route
  - **Access**: Operator role only
  - **Restrictions**: My_fleet vessels only
  - **Input**: vessel_id, destination, fuel_budget (optional)
  - **Output**: Distance, speeds, fuel consumption, costs, savings, AI analysis

### Fleet Management
- `POST /api/fleet/assign` - Assign vessels to fleets
  - **Input**: ship_type (70-79), my_fleet_count, competing_fleet_count
  - **Output**: Assignment results and statistics

### Issue Reporting
- `POST /api/issues/report` - Report operational issue with AI analysis
  - **Access**: Captain role only
  - **Input**: vessel_id, issue_type, description
  - **Output**: Issue ID, severity, impact analysis

### Chatbot
- `POST /api/chatbot/query` - AI-powered maritime queries
  - **Access**: Operator role only
  - **Input**: Natural language query
  - **Output**: AI-generated response with data

## Security & Authentication

### Authentication Model
- Session-based authentication (no JWT tokens)
- Server-side session storage
- Secure password hashing with Werkzeug
- Session cookies with HTTPONLY and Secure flags
- 7-day session lifetime

### Role-Based Access Control (RBAC)

**Operator Role:**
- Full fleet visibility
- Route optimization (my_fleet only)
- Fleet management
- Chatbot access
- Dashboard access

**Captain Role:**
- Issue reporting
- Vessel status viewing
- Limited to assigned vessel

**Port Agent Role:**
- Vessel arrival information
- Port coordination
- Limited vessel tracking

### Data Security
- API keys stored in environment variables
- No secrets committed to repository
- Secure session management
- Input validation and sanitization
- SQL injection protection via SQLAlchemy ORM

## External Integrations

### 1. BarentsWatch AIS API

**Purpose**: Real-time vessel position data  
**Coverage**: Norwegian coastal waters (4,300+ vessels)  
**Authentication**: OAuth 2.0 client credentials flow  
**Rate Limits**: Token expires after 3600 seconds  
**Endpoint**: `https://live.ais.barentswatch.no/live/v1/latest/combined`

**Data Quality:**
- Real-time AIS transponder data
- Update frequency: ~15 minutes per vessel
- Accuracy: GPS-grade positioning
- Coverage: Cargo vessels (types 70-79) represent ~33 vessels in Norwegian waters

**Integration Features:**
- Automatic token refresh
- Smart fallback to demo data if API unavailable
- Unlimited vessel retrieval (no pagination limits)
- Error handling and retry logic

### 2. Google Gemini AI

**Purpose**: Route optimization, issue analysis, chatbot  
**Model**: gemini-2.0-flash (latest as of Nov 2025)  
**Authentication**: API key via environment variable  
**Cost**: Free tier available

**Use Cases:**
- Route optimization rationale and risk assessment
- Issue impact analysis and recommendations
- Maritime query answering
- Distance calculations and suggestions

**Integration Features:**
- JSON structured outputs
- Prompt engineering for maritime domain
- Fallback demo responses when API unavailable
- Context-aware query handling

### 3. Open-Meteo Weather API

**Purpose**: Weather forecasting for vessels and ports  
**Coverage**: Global  
**Authentication**: None required (free API)  
**Rate Limits**: 10,000 calls/day  
**Endpoint**: `https://api.open-meteo.com/v1/`

**Data Provided:**
- Current conditions (temperature, wind, weather codes)
- Marine data (wave height/direction/period, swell)
- 3-day forecast with daily predictions
- Hourly updates

**Integration Features:**
- No authentication overhead
- High reliability
- Global coverage
- Marine-specific parameters

## Deployment Considerations

### Environment Variables Required

```bash
# Database
DATABASE_URL=sqlite:///maritime_ops.db

# Security
SECRET_KEY=<strong-random-key>

# BarentsWatch AIS
BARENTSWATCH_CLIENT_ID=<client-id>
BARENTSWATCH_CLIENT_SECRET=<client-secret>

# Google Gemini AI (optional, has fallback)
GEMINI_API_KEY=<api-key>
```

### Production Configuration

**Web Server:**
- Gunicorn with 2+ workers
- Bind to 0.0.0.0:5000
- Enable port reuse (--reuse-port)
- Worker timeout: 120 seconds

**Database:**
- SQLite in WAL mode (for moderate concurrency)
- Consider PostgreSQL for high-traffic scenarios
- Regular backups recommended
- Database file location: instance/maritime_ops.db

**Performance:**
- Auto-refresh: 30 seconds (configurable)
- AIS cache: 5 minutes
- Connection pooling enabled
- Pre-ping for stale connections

**Monitoring:**
- Application logs via Gunicorn
- Database query performance
- API call limits tracking
- Error rate monitoring

### Scalability Notes

**Current Capacity:**
- Handles 4,300+ vessels without performance issues
- 1,273 active cargo vessels
- Concurrent users: 10-50 (single Gunicorn instance)
- Database: SQLite suitable for <100 concurrent writes

**Scaling Options:**
- Add Gunicorn workers for more concurrent requests
- Migrate to PostgreSQL for higher concurrency
- Implement Redis for session storage
- Add load balancer for multiple instances
- Cache vessel data with Redis/Memcached
- CDN for static assets

## Known Limitations

1. **AIS Data Coverage**: Limited to Norwegian coastal waters via BarentsWatch
2. **Cargo Vessel Density**: Only ~33 cargo vessels in Norwegian waters (fishing/passenger vessels dominate)
3. **Database Concurrency**: SQLite suitable for moderate traffic; PostgreSQL recommended for production
4. **Weather API Limits**: 10,000 calls/day (sufficient for current usage)
5. **Route Optimization**: Restricted to 25 Northern European ports
6. **AI Features**: Requires Gemini API key for full functionality (has fallback mode)

## Future Enhancement Opportunities

1. **Global AIS Coverage**: Integrate additional AIS data providers
2. **Advanced Analytics**: Historical trend analysis, predictive maintenance
3. **Mobile Applications**: iOS/Android apps for captains
4. **Real-time Notifications**: Email/SMS alerts for critical events
5. **Multi-language Support**: Internationalization for global operations
6. **Advanced Routing**: Integration with weather routing algorithms
7. **Crew Management**: Certification tracking, scheduling
8. **Maintenance Tracking**: Planned maintenance, dry dock scheduling
9. **Cargo Tracking**: Integration with cargo manifests
10. **Port Integration**: Direct communication with port systems

## Conclusion

The Maritime Operations Hub represents a modern, AI-enhanced approach to maritime fleet management. By combining real-time AIS tracking, intelligent route optimization, and comprehensive weather data, the system provides maritime stakeholders with the tools needed for efficient, cost-effective operations. The modular architecture and use of industry-standard technologies ensure maintainability and scalability for future growth.
# Home-exam-MAN-5244
