# changes
#createRidesPage.jsx:
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

function CreateRidesPage() {
  const navigate = useNavigate();
  const { user } = useAuth();

  const [formData, setFormData] = useState({
    pickupLocation: '',
    dropoffLocation: '',
    availableSeats: ''
  });

  const handleChange = (e) => {
    setFormData(prev => ({
      ...prev,
      [e.target.name]: e.target.value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();

    if (!user || !user._id) {
      alert('You must be logged in to create a ride.');
      return;
    }

    const rideData = {
      driver: user._id,
      pickupLocation: formData.pickupLocation,
      dropoffLocation: formData.dropoffLocation,
      availableSeats: Number(formData.availableSeats),
      status: 'pending' // lowercase 'pending' to match enum
    };
    const token = localStorage.getItem('token');
    try {
      const res = await fetch('https://smr-qsfr.onrender.com/api/rides/create', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authentication':`Bearer ${token}`
        },
        body: JSON.stringify(rideData)
      });

      if (!res.ok) {
        throw new Error('Ride creation failed');
      }

      alert('Ride created successfully!');
      navigate('/rides');
    } catch (err) {
      console.error(err);
      alert('Failed to create ride');
    }
  };

  return (
    <div className="max-w-xl mx-auto p-6 bg-white rounded shadow-md mt-20">
      <h2 className="text-3xl font-bold text-center mb-6 text-orange-600">Create a New Ride</h2>
      <form onSubmit={handleSubmit} className="space-y-5">
        <input
          type="text"
          name="pickupLocation"
          placeholder="Pickup Location"
          className="w-full p-3 border rounded"
          value={formData.pickupLocation}
          onChange={handleChange}
          required
        />
        <input
          type="text"
          name="dropoffLocation"
          placeholder="Dropoff Location"
          className="w-full p-3 border rounded"
          value={formData.dropoffLocation}
          onChange={handleChange}
          required
        />
        <input
          type="number"
          name="availableSeats"
          placeholder="Available Seats"
          className="w-full p-3 border rounded"
          value={formData.availableSeats}
          onChange={handleChange}
          required
          min={1}
        />
        <button
          type="submit"
          className="w-full bg-orange-500 text-white py-3 rounded hover:bg-orange-600 transition"
        >
          Create Ride
        </button>
      </form>
    </div>
  );
}

export default CreateRidesPage;

#AuthContext.jsx:
import React, { createContext, useContext, useState, useEffect } from 'react';
import { useLocalStorage } from '../hooks/UseLocalStorage';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [token, setToken] = useLocalStorage('token', null);
  const [storedUser, setStoredUser] = useLocalStorage('user', null);
  const [user, setUser] = useState(storedUser);
  const [isAuthenticated, setIsAuthenticated] = useState(!!storedUser && !!token);

  useEffect(() => {
    const validateTokenAndFetchProfile = async () => {
      if (token) {
        try {
          const res = await fetch('https://smr-qsfr.onrender.com/api/users/profile', {
            headers: { Authorization: `Bearer ${token}` },
          });

          if (res.ok) {
            const data = await res.json();
            if (data && data._id) {
              setUser(data);
              setStoredUser(data);
              setIsAuthenticated(true);
            } else {
              logout();
            }
          } else if (res.status === 401) {
            // Token is expired or invalid
            logout();
          } else {
            // Some other error
            console.error('Unexpected error during token validation');
            logout();
          }
        } catch (error) {
          console.error('Error validating token:', error);
          logout();
        }
      } else {
        logout();
      }
    };

    validateTokenAndFetchProfile();
  }, [token]);

  const logout = () => {
    setToken(null);
    setUser(null);
    setStoredUser(null);
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider
      value={{
        token,
        user,
        isAuthenticated,
        setIsAuthenticated,
        setToken,
        setUser,
        logout,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);

#LoginPage.jsx: 
/* eslint-disable no-unused-vars */
// src/pages/LoginPage.jsx
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

const LoginPage = () => {
  const { setToken, setUser, setIsAuthenticated } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      // 1. Login and get token
      const response = await fetch('https://smr-qsfr.onrender.com/api/users/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      });

      const data = await response.json();
     
      console.log("Status:", response.status);
      console.log("Response data:", data);

      if (!response.ok) {
        setError(data.message || 'Login failed');
        return;
      }

      const token = data.token;
      setToken(token);

      // 2. Fetch user profile with token
      const profileRes = await fetch('https://smr-qsfr.onrender.com/api/users/profile', {
        headers: { Authorization: `Bearer ${token}` },
      });

      const userData = await profileRes.json();

      if (profileRes.ok && userData._id) {
        setUser(userData);
        setIsAuthenticated(true);
        navigate('/');
      } else {
        setError('Failed to retrieve user profile.');
        setToken(null);
      }
    } catch (err) {
      console.error(err);
      setError('An error occurred. Please try again.');
    }
  };

  return (
    <div className="flex items-center justify-center min-h-screen bg-gray-50">
      <div className="w-full max-w-md p-8 bg-white rounded-lg shadow-md">
        <h2 className="text-2xl font-bold text-center text-orange-500 mb-6">Login to Your Account</h2>
        {error && <div className="text-red-500 text-sm mb-4">{error}</div>}
        <form onSubmit={handleLogin} className="space-y-4">
          <div>
            <label className="block text-sm font-semibold text-gray-600">Email</label>
            <input 
              type="email" 
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required 
              className="w-full px-4 py-2 mt-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-orange-400"
            />
          </div>
          <div>
            <label className="block text-sm font-semibold text-gray-600">Password</label>
            <input 
              type="password" 
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required 
              className="w-full px-4 py-2 mt-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-orange-400"
            />
          </div>
          <button 
            type="submit" 
            className="w-full bg-orange-500 hover:bg-orange-600 text-white py-2 rounded-md font-semibold transition"
          >
            Login
          </button>
        </form>
        <p className="mt-4 text-sm text-center text-gray-600">
          Don't have an account? <a href="/register" className="text-orange-500 hover:underline">Register</a>
        </p>
      </div>
    </div>
  );
};

export default LoginPage;
