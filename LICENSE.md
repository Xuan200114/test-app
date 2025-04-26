import React, { useState, useEffect } from 'react';
import {
  SafeAreaView,
  StyleSheet,
  View,
  Text,
  TouchableOpacity,
  StatusBar,
  TextInput,
  Alert,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import Sound from 'react-native-sound';
import LottieView from 'lottie-react-native';
import moment from 'moment';
import axios from 'axios';
import Modal from 'react-native-modal';
import Slider from '@react-native-community/slider';
import RNPickerSelect from 'react-native-picker-select';
import Geolocation from '@react-native-community/geolocation';
import Icon from 'react-native-vector-icons/MaterialCommunityIcons';

const WEATHER_API_KEY = 'YOUR_WEATHER_API_KEY'; // 需要替換為實際的 API key

const App = () => {
  const [isWorking, setIsWorking] = useState(false);
  const [earnings, setEarnings] = useState(0);
  const [showCoinRain, setShowCoinRain] = useState(false);
  const [startTime, setStartTime] = useState(null);
  const [hourlyRate, setHourlyRate] = useState(200);
  const [workDuration, setWorkDuration] = useState(8);
  const [weather, setWeather] = useState(null);
  const [mood, setMood] = useState('😊');
  const [isSettingsVisible, setSettingsVisible] = useState(false);
  const [customSound, setCustomSound] = useState('');

  // 心情選項
  const moodOptions = [
    { label: '開心 😊', value: '😊' },
    { label: '普通 😐', value: '😐' },
    { label: '疲憊 😫', value: '😫' },
    { label: '興奮 🤩', value: '🤩' },
    { label: '專注 🧐', value: '🧐' },
  ];

  useEffect(() => {
    loadSettings();
    getWeather();
  }, []);

  useEffect(() => {
    let timer;
    if (isWorking) {
      const secondRate = hourlyRate / 3600;
      timer = setInterval(() => {
        const newEarnings = earnings + secondRate;
        setEarnings(newEarnings);
        
        if (Math.floor(newEarnings) > Math.floor(earnings)) {
          setShowCoinRain(true);
          setTimeout(() => setShowCoinRain(false), 3000);
          playSound();
        }
      }, 1000);
    }
    return () => clearInterval(timer);
  }, [isWorking, earnings, hourlyRate]);

  const loadSettings = async () => {
    try {
      const settings = await AsyncStorage.getItem('settings');
      if (settings) {
        const { savedHourlyRate, savedWorkDuration, savedCustomSound } = JSON.parse(settings);
        setHourlyRate(savedHourlyRate || 200);
        setWorkDuration(savedWorkDuration || 8);
        setCustomSound(savedCustomSound || '');
      }
    } catch (error) {
      console.log('載入設置失敗', error);
    }
  };

  const saveSettings = async () => {
    try {
      const settings = {
        savedHourlyRate: hourlyRate,
        savedWorkDuration: workDuration,
        savedCustomSound: customSound,
      };
      await AsyncStorage.setItem('settings', JSON.stringify(settings));
      setSettingsVisible(false);
    } catch (error) {
      console.log('保存設置失敗', error);
    }
  };

  const getWeather = () => {
    Geolocation.getCurrentPosition(
      async position => {
        try {
          const { latitude, longitude } = position.coords;
          const response = await axios.get(
            `https://api.openweathermap.org/data/2.5/weather?lat=${latitude}&lon=${longitude}&appid=${WEATHER_API_KEY}&units=metric`
          );
          setWeather(response.data);
        } catch (error) {
          console.log('獲取天氣失敗', error);
        }
      },
      error => console.log('獲取位置失敗', error),
      { enableHighAccuracy: true, timeout: 20000, maximumAge: 1000 }
    );
  };

  const playSound = () => {
    const soundFile = customSound || 'bell.mp3';
    const bell = new Sound(soundFile, Sound.MAIN_BUNDLE, (error) => {
      if (error) {
        console.log('播放音效失敗', error);
        return;
      }
      bell.play();
    });
  };

  const toggleWork = () => {
    if (!isWorking) {
      setStartTime(new Date());
    } else {
      saveWorkSession();
    }
    setIsWorking(!isWorking);
  };

  const saveWorkSession = async () => {
    try {
      const endTime = new Date();
      const duration = moment.duration(moment(endTime).diff(moment(startTime)));
      const session = {
        date: moment().format('YYYY-MM-DD'),
        startTime: startTime.toISOString(),
        endTime: endTime.toISOString(),
        duration: duration.asHours(),
        earnings: earnings,
        mood: mood,
      };
      
      const existingSessions = await AsyncStorage.getItem('workSessions');
      const sessions = existingSessions ? JSON.parse(existingSessions) : [];
      sessions.push(session);
      await AsyncStorage.setItem('workSessions', JSON.stringify(sessions));
    } catch (error) {
      console.log('保存工作記錄失敗', error);
    }
  };

  const renderWeather = () => {
    if (!weather) return null;
    return (
      <View style={styles.weatherContainer}>
        <Text style={styles.weatherText}>
          {weather.main.temp}°C {weather.weather[0].main}
        </Text>
        <Icon name={getWeatherIcon(weather.weather[0].main)} size={24} color="#fff" />
      </View>
    );
  };

  const getWeatherIcon = (condition) => {
    const icons = {
      Clear: 'weather-sunny',
      Clouds: 'weather-cloudy',
      Rain: 'weather-rainy',
      Snow: 'weather-snowy',
      default: 'weather-partly-cloudy',
    };
    return icons[condition] || icons.default;
  };

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="light-content" />
      <View style={styles.header}>
        {renderWeather()}
        <TouchableOpacity onPress={() => setSettingsVisible(true)}>
          <Icon name="cog" size={24} color="#fff" />
        </TouchableOpacity>
      </View>
      <View style={styles.content}>
        <View style={styles.moodContainer}>
          <Text style={styles.moodText}>今日心情：{mood}</Text>
          <RNPickerSelect
            onValueChange={(value) => setMood(value)}
            items={moodOptions}
            value={mood}
            style={pickerSelectStyles}
          />
        </View>
        <Text style={styles.earnings}>
          ¥ {earnings.toFixed(2)}
        </Text>
        <TouchableOpacity
          style={[styles.button, isWorking ? styles.stopButton : styles.startButton]}
          onPress={toggleWork}
        >
          <Text style={styles.buttonText}>
            {isWorking ? '下班' : '上班'}
          </Text>
        </TouchableOpacity>
        {showCoinRain && (
          <LottieView
            source={require('./assets/coin-rain.json')}
            autoPlay
            loop={false}
            style={styles.coinRain}
          />
        )}
      </View>

      <Modal
        isVisible={isSettingsVisible}
        onBackdropPress={() => setSettingsVisible(false)}
        style={styles.modal}
      >
        <View style={styles.modalContent}>
          <Text style={styles.modalTitle}>設置</Text>
          <View style={styles.settingItem}>
            <Text>時薪 (¥/小時)</Text>
            <TextInput
              style={styles.input}
              value={String(hourlyRate)}
              onChangeText={(text) => setHourlyRate(Number(text) || 0)}
              keyboardType="numeric"
            />
          </View>
          <View style={styles.settingItem}>
            <Text>工作時長 (小時)</Text>
            <Slider
              style={styles.slider}
              minimumValue={1}
              maximumValue={12}
              step={0.5}
              value={workDuration}
              onValueChange={setWorkDuration}
            />
            <Text>{workDuration} 小時</Text>
          </View>
          <View style={styles.settingItem}>
            <Text>自定義鈴聲 URL</Text>
            <TextInput
              style={styles.input}
              value={customSound}
              onChangeText={setCustomSound}
              placeholder="輸入音樂文件 URL"
            />
          </View>
          <TouchableOpacity style={styles.saveButton} onPress={saveSettings}>
            <Text style={styles.saveButtonText}>保存</Text>
          </TouchableOpacity>
        </View>
      </Modal>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#1a1a1a',
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 15,
  },
  weatherContainer: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  weatherText: {
    color: '#fff',
    marginRight: 10,
  },
  content: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  moodContainer: {
    marginBottom: 20,
    alignItems: 'center',
  },
  moodText: {
    color: '#fff',
    fontSize: 18,
    marginBottom: 10,
  },
  earnings: {
    fontSize: 48,
    fontWeight: 'bold',
    color: '#00ff00',
    marginBottom: 40,
  },
  button: {
    paddingHorizontal: 40,
    paddingVertical: 15,
    borderRadius: 25,
    elevation: 5,
  },
  startButton: {
    backgroundColor: '#00ff00',
  },
  stopButton: {
    backgroundColor: '#ff3b30',
  },
  buttonText: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#ffffff',
  },
  coinRain: {
    position: 'absolute',
    width: '100%',
    height: '100%',
  },
  modal: {
    justifyContent: 'flex-end',
    margin: 0,
  },
  modalContent: {
    backgroundColor: 'white',
    padding: 20,
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
  },
  modalTitle: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  settingItem: {
    marginBottom: 20,
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 10,
    marginTop: 5,
  },
  slider: {
    width: '100%',
    height: 40,
  },
  saveButton: {
    backgroundColor: '#00ff00',
    padding: 15,
    borderRadius: 8,
    alignItems: 'center',
  },
  saveButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
  },
});

const pickerSelectStyles = StyleSheet.create({
  inputIOS: {
    fontSize: 16,
    paddingVertical: 12,
    paddingHorizontal: 10,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    color: '#fff',
    paddingRight: 30,
  },
  inputAndroid: {
    fontSize: 16,
    paddingHorizontal: 10,
    paddingVertical: 8,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    color: '#fff',
    paddingRight: 30,
  },
});

export default App; 
