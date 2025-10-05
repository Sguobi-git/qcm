import React, { useState, useEffect, useRef } from 'react';
import { Upload, Clock, CheckCircle, XCircle, ChevronRight, RotateCcw } from 'lucide-react';

const MCQApp = () => {
  const [questions, setQuestions] = useState([]);
  const [currentIndex, setCurrentIndex] = useState(0);
  const [selectedAnswers, setSelectedAnswers] = useState([]);
  const [showCorrection, setShowCorrection] = useState(false);
  const [activeTime, setActiveTime] = useState(0);
  const [isActive, setIsActive] = useState(false);
  const [results, setResults] = useState([]);
  const [showSummary, setShowSummary] = useState(false);
  const timerRef = useRef(null);

  // Timer logic - only counts when answering
  useEffect(() => {
    if (isActive && !showCorrection) {
      timerRef.current = setInterval(() => {
        setActiveTime(prev => prev + 1);
      }, 1000);
    } else {
      clearInterval(timerRef.current);
    }
    return () => clearInterval(timerRef.current);
  }, [isActive, showCorrection]);

  const parseQCMText = (text) => {
    const questionBlocks = text.split(/Question\s+n?°?\s*\d+\s*:/i).filter(block => block.trim());
    const parsed = [];

    questionBlocks.forEach((block, idx) => {
      const lines = block.split('\n').filter(line => line.trim());
      const options = [];
      let currentOption = null;

      lines.forEach(line => {
        const optionMatch = line.match(/^([A-E])\.\s*(VRAI|FAUX|Vrai|Faux)/i);
        if (optionMatch) {
          if (currentOption) options.push(currentOption);
          currentOption = {
            letter: optionMatch[1],
            isCorrect: optionMatch[2].toUpperCase() === 'VRAI',
            explanation: line.substring(optionMatch[0].length).trim().replace(/^[.:]\s*/, '')
          };
        } else if (currentOption && line.trim()) {
          currentOption.explanation += ' ' + line.trim();
        }
      });
      if (currentOption) options.push(currentOption);

      // Handle "Réponse VRAIE : X" format
      const singleAnswerMatch = block.match(/Réponse\s+VRAIE?\s*:\s*([A-E])/i);
      if (singleAnswerMatch && options.length === 0) {
        ['A', 'B', 'C', 'D', 'E'].forEach(letter => {
          options.push({
            letter,
            isCorrect: letter === singleAnswerMatch[1],
            explanation: letter === singleAnswerMatch[1] 
              ? block.split('\n').slice(1).join(' ').trim()
              : ''
          });
        });
      }

      if (options.length > 0) {
        parsed.push({
          number: idx + 1,
          options,
          text: block.split('\n')[0].trim()
        });
      }
    });

    return parsed;
  };

  const handleFileUpload = (event) => {
    const file = event.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const text = e.target.result;
      const parsedQuestions = parseQCMText(text);
      setQuestions(parsedQuestions);
      setCurrentIndex(0);
      setActiveTime(0);
      setIsActive(true);
      setResults([]);
      setShowSummary(false);
    };
    reader.readAsText(file);
  };

  const handleAnswerToggle = (letter) => {
    if (showCorrection) return;
    
    setSelectedAnswers(prev => 
      prev.includes(letter) 
        ? prev.filter(l => l !== letter)
        : [...prev, letter]
    );
  };

  const handleValidate = () => {
    const current = questions[currentIndex];
    const correctAnswers = current.options.filter(opt => opt.isCorrect).map(opt => opt.letter);
    const isCorrect = 
      selectedAnswers.length === correctAnswers.length &&
      selectedAnswers.every(ans => correctAnswers.includes(ans));

    setResults(prev => [...prev, {
      questionNumber: current.number,
      correct: isCorrect,
      selectedAnswers: [...selectedAnswers],
      correctAnswers
    }]);
    
    setShowCorrection(true);
  };

  const handleNext = () => {
    if (currentIndex < questions.length - 1) {
      setCurrentIndex(prev => prev + 1);
      setSelectedAnswers([]);
      setShowCorrection(false);
    } else {
      setShowSummary(true);
      setIsActive(false);
    }
  };

  const handleReset = () => {
    setQuestions([]);
    setCurrentIndex(0);
    setSelectedAnswers([]);
    setShowCorrection(false);
    setActiveTime(0);
    setIsActive(false);
    setResults([]);
    setShowSummary(false);
  };

  const formatTime = (seconds) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };

  if (showSummary) {
    const score = results.filter(r => r.correct).length;
    const total = results.length;
    const percentage = Math.round((score / total) * 100);

    return (
      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
        <div className="max-w-2xl mx-auto">
          <div className="bg-white rounded-2xl shadow-xl p-8">
            <h2 className="text-3xl font-bold text-center mb-6 text-indigo-900">
              Résumé de la session
            </h2>
            
            <div className="grid grid-cols-2 gap-4 mb-8">
              <div className="bg-indigo-50 rounded-xl p-6 text-center">
                <div className="text-4xl font-bold text-indigo-600">{score}/{total}</div>
                <div className="text-gray-600 mt-2">Questions correctes</div>
              </div>
              <div className="bg-blue-50 rounded-xl p-6 text-center">
                <div className="text-4xl font-bold text-blue-600">{percentage}%</div>
                <div className="text-gray-600 mt-2">Score</div>
              </div>
            </div>

            <div className="bg-gray-50 rounded-xl p-6 mb-6">
              <div className="flex items-center justify-center gap-2 text-xl">
                <Clock className="text-gray-600" />
                <span className="font-semibold text-gray-700">
                  Temps total: {formatTime(activeTime)}
                </span>
              </div>
              <div className="text-center text-gray-600 mt-2">
                Temps moyen par question: {formatTime(Math.round(activeTime / total))}
              </div>
            </div>

            <button
              onClick={handleReset}
              className="w-full bg-indigo-600 text-white py-4 rounded-xl font-semibold hover:bg-indigo-700 transition flex items-center justify-center gap-2"
            >
              <RotateCcw size={20} />
              Nouvelle session
            </button>
          </div>
        </div>
      </div>
    );
  }

  if (questions.length === 0) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
        <div className="bg-white rounded-2xl shadow-xl p-8 max-w-md w-full">
          <div className="text-center mb-6">
            <h1 className="text-3xl font-bold text-indigo-900 mb-2">
              QCM Médical
            </h1>
            <p className="text-gray-600">
              Importez votre fichier pour commencer
            </p>
          </div>

          <label className="flex flex-col items-center justify-center w-full h-48 border-2 border-dashed border-indigo-300 rounded-xl cursor-pointer hover:border-indigo-500 hover:bg-indigo-50 transition">
            <Upload className="text-indigo-400 mb-3" size={48} />
            <span className="text-indigo-600 font-semibold mb-1">
              Cliquez pour importer
            </span>
            <span className="text-gray-500 text-sm">
              Fichiers .txt ou .docx
            </span>
            <input
              type="file"
              className="hidden"
              accept=".txt,.docx,.doc"
              onChange={handleFileUpload}
            />
          </label>

          <div className="mt-6 p-4 bg-blue-50 rounded-xl">
            <p className="text-sm text-gray-700">
              <strong>Comment préparer vos fichiers:</strong>
              <br />1. Ouvrez votre PDF dans Google Docs
              <br />2. Téléchargez en format Texte (.txt)
              <br />3. Importez le fichier ici
            </p>
          </div>
        </div>
      </div>
    );
  }

  const current = questions[currentIndex];
  const progress = ((currentIndex + 1) / questions.length) * 100;

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="max-w-2xl mx-auto">
        {/* Header */}
        <div className="bg-white rounded-t-2xl shadow-lg p-4">
          <div className="flex justify-between items-center mb-3">
            <div className="flex items-center gap-2">
              <Clock className="text-indigo-600" size={20} />
              <span className="font-semibold text-gray-700">
                {formatTime(activeTime)}
              </span>
            </div>
            <div className="text-sm font-medium text-gray-600">
              Question {currentIndex + 1} / {questions.length}
            </div>
          </div>
          <div className="w-full bg-gray-200 rounded-full h-2">
            <div
              className="bg-indigo-600 h-2 rounded-full transition-all duration-300"
              style={{ width: `${progress}%` }}
            />
          </div>
        </div>

        {/* Question */}
        <div className="bg-white shadow-lg p-6">
          <h2 className="text-xl font-bold text-gray-800 mb-6">
            Question {current.number}
          </h2>

          <div className="space-y-3">
            {current.options.map((option) => {
              const isSelected = selectedAnswers.includes(option.letter);
              const showResult = showCorrection;
              const isCorrectAnswer = option.isCorrect;

              return (
                <div key={option.letter}>
                  <button
                    onClick={() => handleAnswerToggle(option.letter)}
                    disabled={showCorrection}
                    className={`w-full text-left p-4 rounded-xl border-2 transition ${
                      showResult
                        ? isCorrectAnswer
                          ? 'bg-green-50 border-green-500'
                          : isSelected
                          ? 'bg-red-50 border-red-500'
                          : 'bg-gray-50 border-gray-300'
                        : isSelected
                        ? 'bg-indigo-50 border-indigo-500'
                        : 'bg-white border-gray-300 hover:border-indigo-300'
                    }`}
                  >
                    <div className="flex items-start gap-3">
                      <div className={`flex-shrink-0 w-6 h-6 rounded border-2 flex items-center justify-center ${
                        showResult
                          ? isCorrectAnswer
                            ? 'border-green-500 bg-green-500'
                            : isSelected
                            ? 'border-red-500 bg-red-500'
                            : 'border-gray-300'
                          : isSelected
                          ? 'border-indigo-500 bg-indigo-500'
                          : 'border-gray-400'
                      }`}>
                        {showResult ? (
                          isCorrectAnswer ? (
                            <CheckCircle size={16} className="text-white" />
                          ) : isSelected ? (
                            <XCircle size={16} className="text-white" />
                          ) : null
                        ) : isSelected ? (
                          <CheckCircle size={16} className="text-white" />
                        ) : null}
                      </div>
                      <div className="flex-1">
                        <span className="font-semibold">{option.letter}.</span>
                        {showResult && option.explanation && (
                          <p className="mt-2 text-sm text-gray-700">
                            {option.explanation}
                          </p>
                        )}
                      </div>
                    </div>
                  </button>
                </div>
              );
            })}
          </div>
        </div>

        {/* Actions */}
        <div className="bg-white rounded-b-2xl shadow-lg p-4">
          {!showCorrection ? (
            <button
              onClick={handleValidate}
              disabled={selectedAnswers.length === 0}
              className="w-full bg-indigo-600 text-white py-4 rounded-xl font-semibold hover:bg-indigo-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition"
            >
              Valider
            </button>
          ) : (
            <button
              onClick={handleNext}
              className="w-full bg-green-600 text-white py-4 rounded-xl font-semibold hover:bg-green-700 transition flex items-center justify-center gap-2"
            >
              {currentIndex < questions.length - 1 ? (
                <>
                  Question suivante
                  <ChevronRight size={20} />
                </>
              ) : (
                'Voir le résumé'
              )}
            </button>
          )}
        </div>
      </div>
    </div>
  );
};

export default MCQApp;
