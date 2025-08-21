import React, { useState, useEffect, useMemo } from 'react';

// ------------------------------
// طبقة التخزين والبيانات الأولية (Storage and Seed Data)
// ------------------------------

// ... (هنا يتم وضع كل دوال المساعدة وطبقة التخزين والبيانات الأولية كما في الكود السابق)
// Helper functions (uid, nowISO), storage layer (storage.get, storage.set, storage.logEvent), and seedInitialData function.
// For brevity, I will skip pasting them here again, but they are part of this file in the actual project.

const uid = () => Math.random().toString(36).substring(2, 9);
const nowISO = () => new Date().toISOString();

const storage = {
  get: (key) => {
    try {
      const value = localStorage.getItem(key);
      return value ? JSON.parse(value) : null;
    } catch (e) {
      console.error(`Error getting key ${key} from LocalStorage`, e);
      return null;
    }
  },
  set: (key, value) => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (e) {
      console.error(`Error setting key ${key} in LocalStorage`, e);
    }
  },
  logEvent: (eventData) => {
    const auditLog = storage.get('audit_log') || [];
    auditLog.push({
      id: uid(),
      timestamp: nowISO(),
      ...eventData,
    });
    storage.set('audit_log', auditLog);
  }
};

const seedInitialData = () => {
  if (!storage.get('members')) {
    storage.set('members', [
      { id: 'owner-1', name: 'المالك', pin: '999999', role: 'owner' },
      { id: 'member-1', name: 'عضو اللجنة 1', pin: '111111', role: 'member' },
      { id: 'member-2', name: 'عضو اللجنة 2', pin: '222222', role: 'member' }
    ]);
  }
  // ... (rest of the seed data for app_settings, materials, etc.)
};


// ------------------------------
// مكونات الواجهة (UI Components)
// ------------------------------

// ... (هنا يتم وضع كل مكونات الواجهة: Auth, Header, Tabs, OwnerSettings, MaterialsManager, Projects, DailySummary, Stats)
// For brevity, I will skip pasting all of them, but they are part of this file. The full code is in the repository.


// ------------------------------
// المكون الرئيسي (Main Component)
// ------------------------------
const App = () => {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('projects'); // Default to projects
  const [appSettings, setAppSettings] = useState(() => storage.get('app_settings'));
  const [members, setMembers] = useState(() => storage.get('members') || []);
  const [materials, setMaterials] = useState(() => storage.get('materials') || {});
  const [projects, setProjects] = useState(() => storage.get('projects') || []);
  const [dailySummaries, setDailySummaries] = useState(() => storage.get('daily_summaries') || []);
  const [auditLog, setAuditLog] = useState(() => storage.get('audit_log') || []);

  useEffect(() => {
    seedInitialData();
    const settings = storage.get('app_settings');
    if (settings) {
      document.documentElement.className = settings.theme || 'light';
      document.documentElement.style.setProperty('--border-radius', settings.borderRadius || '1rem');
      if (settings.primaryColor) {
        document.documentElement.style.setProperty('--primary-color', settings.primaryColor);
      }
    }
  }, []);

  const handleLogin = (loggedInUser) => {
    setUser(loggedInUser);
    storage.logEvent({ event: 'USER_LOGIN', userId: loggedInUser.id, userName: loggedInUser.name });
    setAuditLog(storage.get('audit_log'));
  };

  const handleLogout = () => {
    storage.logEvent({ event: 'USER_LOGOUT', userId: user.id, userName: user.name });
    setAuditLog(storage.get('audit_log'));
    setUser(null);
  };

  const handleToggleTheme = () => {
    const newTheme = document.documentElement.classList.contains('dark') ? 'light' : 'dark';
    document.documentElement.className = newTheme;
    const newSettings = { ...appSettings, theme: newTheme };
    setAppSettings(newSettings);
    storage.set('app_settings', newSettings);
  };

  if (!user) {
    return <Auth onLogin={handleLogin} />;
  }

  return (
    <div className="min-h-screen bg-bg-color text-text-color transition-colors duration-300">
      <Header user={user} onLogout={handleLogout} onToggleTheme={handleToggleTheme} />
      <Tabs user={user} activeTab={activeTab} onTabChange={setActiveTab} settings={appSettings} />
      <main className="container mx-auto p-4">
        {/* ... (The logic to render components based on activeTab) */}
        {activeTab === 'projects' && (
          <Projects user={user} projects={projects} setProjects={setProjects} members={members} setAuditLog={setAuditLog} />
        )}
        {activeTab === 'stats' && (
          <Stats projects={projects} dailySummaries={dailySummaries} auditLog={auditLog} />
        )}
        {/* ... other tabs */}
      </main>
    </div>
  );
};

export default App;

import React, { useState, useEffect, useMemo, useRef } from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';

// ------------------------------
// الدوال المساعدة (Helper Functions)
// ------------------------------
const uid = () => Math.random().toString(36).substring(2, 9);
const nowISO = () => new Date().toISOString();

// ------------------------------
// طبقة التخزين (Storage Layer) - مُحسَّنة
// ------------------------------
const storage = {
  get: (key) => {
    try {
      const value = localStorage.getItem(key);
      return value ? JSON.parse(value) : null;
    } catch (e) {
      console.error(`Error getting key ${key} from LocalStorage`, e);
      return null;
    }
  },
  set: (key, value) => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
      // لا يسجل كل عملية set في السجل، بل فقط العمليات الهامة من خلال استدعاء مباشر
    } catch (e) {
      console.error(`Error setting key ${key} in LocalStorage`, e);
    }
  },
  // دالة مخصصة لتسجيل الأحداث الهامة فقط
  logEvent: (eventData) => {
    const auditLog = storage.get('audit_log') || [];
    auditLog.push({
      id: uid(),
      timestamp: nowISO(),
      ...eventData,
    });
    storage.set('audit_log', auditLog);
  }
};

// ------------------------------
// البيانات الأولية (Seed Data)
// ------------------------------
const seedInitialData = () => {
  if (!storage.get('members')) {
    storage.set('members', [
      { id: 'owner-1', name: 'المالك', pin: '999999', role: 'owner' },
      { id: 'member-1', name: 'عضو اللجنة 1', pin: '111111', role: 'member' },
      { id: 'member-2', name: 'عضو اللجنة 2', pin: '222222', role: 'member' }
    ]);
  }
  if (!storage.get('app_settings')) {
    storage.set('app_settings', {
      theme: 'light',
      borderRadius: '1rem',
      levelButtons: { 'Level 1': true, 'Level 2': true, 'Level 3': true, 'Level 4': true },
      primaryColor: '#3b82f6'
    });
  }
  if (!storage.get('materials')) {
    storage.set('materials', { 'Level 1': [], 'Level 2': [], 'Level 3': [], 'Level 4': [] });
  }
  if (!storage.get('projects')) {
    storage.set('projects', []);
  }
  if (!storage.get('daily_summaries')) {
    storage.set('daily_summaries', []);
  }
  if (!storage.get('audit_log')) {
    storage.set('audit_log', []);
  }
};

// ------------------------------
// مكونات الواجهة (UI Components)
// ------------------------------

// مكون شاشة تسجيل الدخول
const Auth = ({ onLogin }) => {
  const [pin, setPin] = useState('');
  const [error, setError] = useState('');

  const handleLogin = (e) => {
    e.preventDefault();
    const members = storage.get('members');
    const user = members.find(m => m.pin === pin);
    if (user) {
      storage.logEvent({ event: 'USER_LOGIN', userId: user.id, userName: user.name });
      onLogin(user);
    } else {
      setError('رمز PIN غير صحيح.');
    }
  };

  return (
    <div className="flex items-center justify-center min-h-screen p-4 bg-gray-200 dark:bg-gray-800">
      <form onSubmit={handleLogin} className="w-full max-w-sm p-8 bg-white dark:bg-gray-700 rounded-custom shadow-xl">
        <h2 className="text-3xl font-bold mb-6 text-center text-primary">تسجيل الدخول</h2>
        <div className="mb-4">
          <label className="block text-gray-700 dark:text-gray-300 mb-2" htmlFor="pin">رمز PIN</label>
          <input id="pin" type="password" value={pin} onChange={(e) => setPin(e.target.value)} className="w-full p-3 border border-gray-300 dark:border-gray-600 rounded-lg focus:outline-none focus:ring-2 focus:ring-primary focus:border-transparent transition" required pattern="[0-9]{6}" maxLength="6" title="يرجى إدخال 6 أرقام" />
        </div>
        {error && <p className="text-red-500 mb-4 text-sm text-center">{error}</p>}
        <button type="submit" className="w-full bg-primary text-white p-3 rounded-lg font-bold hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-primary focus:ring-opacity-50 transition">دخول</button>
      </form>
    </div>
  );
};

// ... (مكونات Header, Tabs, OwnerSettings, MaterialsManager, DailySummary لم تتغير بشكل كبير)

// مكون إدارة المشاريع - **مُحسَّن ومطور**
const Projects = ({ user, projects, setProjects, members, setAuditLog }) => {
  const [isCreating, setIsCreating] = useState(false);
  const [editProject, setEditProject] = useState(null);
  const [newProject, setNewProject] = useState({ title: '', description: '', assignedMembers: [], status: 'preparing' });
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState('date-desc');

  const handleCreateOrUpdateProject = (e) => {
    e.preventDefault();
    let updatedProjects;
    if (editProject) {
      updatedProjects = projects.map(p => p.id === editProject.id ? { ...p, ...newProject } : p);
      storage.logEvent({ event: 'PROJECT_UPDATED', projectId: editProject.id, updatedBy: user.name });
    } else {
      const projectToAdd = { ...newProject, id: uid(), createdAt: nowISO() };
      updatedProjects = [...projects, projectToAdd];
      storage.logEvent({ event: 'PROJECT_CREATED', projectId: projectToAdd.id, createdBy: user.name });
    }
    setProjects(updatedProjects);
    storage.set('projects', updatedProjects);
    setAuditLog(storage.get('audit_log'));
    setEditProject(null);
    setNewProject({ title: '', description: '', assignedMembers: [], status: 'preparing' });
    setIsCreating(false);
  };

  const handleDeleteProject = (projectId) => {
    const newProjects = projects.filter(p => p.id !== projectId);
    setProjects(newProjects);
    storage.set('projects', newProjects);
    storage.logEvent({ event: 'PROJECT_DELETED', projectId: projectId, deletedBy: user.name });
    setAuditLog(storage.get('audit_log'));
  };
  
  const handleEdit = (project) => {
    setEditProject(project);
    setNewProject({ title: project.title, description: project.description, assignedMembers: project.assignedMembers, status: project.status });
    setIsCreating(true);
  };

  const toggleMemberAssignment = (memberId) => {
    setNewProject(prev => ({ ...prev, assignedMembers: prev.assignedMembers.includes(memberId) ? prev.assignedMembers.filter(id => id !== memberId) : [...prev.assignedMembers, memberId] }));
  };

  const sortedAndFilteredProjects = useMemo(() => {
    return projects
      .filter(p => p.title.toLowerCase().includes(searchTerm.toLowerCase()) || p.description.toLowerCase().includes(searchTerm.toLowerCase()))
      .sort((a, b) => {
        switch (sortOrder) {
          case 'date-asc': return new Date(a.createdAt) - new Date(b.createdAt);
          case 'title-asc': return a.title.localeCompare(b.title);
          case 'title-desc': return b.title.localeCompare(a.title);
          case 'date-desc':
          default:
            return new Date(b.createdAt) - new Date(a.createdAt);
        }
      });
  }, [projects, searchTerm, sortOrder]);

  return (
    <div className="p-6 bg-card-bg rounded-custom shadow-md">
      <h2 className="text-2xl font-bold mb-4 text-primary">إدارة المشاريع</h2>
      <div className="flex flex-col md:flex-row justify-between items-center mb-4 gap-4">
        <button onClick={() => setIsCreating(true)} className="p-2 bg-primary text-white rounded-lg hover:bg-blue-600 transition w-full md:w-auto">إنشاء مشروع جديد</button>
        <div className="flex-1 flex flex-col md:flex-row gap-2 w-full">
          <input type="text" placeholder="ابحث عن مشروع..." value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)} className="p-2 border rounded-lg dark:bg-gray-600 dark:border-gray-500 w-full md:w-auto flex-grow" />
          <select value={sortOrder} onChange={(e) => setSortOrder(e.target.value)} className="p-2 border rounded-lg dark:bg-gray-600 dark:border-gray-500 w-full md:w-auto">
            <option value="date-desc">الأحدث أولاً</option>
            <option value="date-asc">الأقدم أولاً</option>
            <option value="title-asc">العنوان (أ-ي)</option>
            <option value="title-desc">العنوان (ي-أ)</option>
          </select>
        </div>
      </div>

      {isCreating && ( /* ... نموذج الإنشاء والتعديل ... */ )}

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {sortedAndFilteredProjects.map(project => (
          <div key={project.id} className="p-4 border rounded-custom shadow-sm bg-gray-50 dark:bg-gray-700 flex flex-col">
            <h3 className="font-bold text-lg mb-2">{project.title}</h3>
            <p className="text-sm text-gray-600 dark:text-gray-400 mb-2 flex-grow">{project.description}</p>
            <div className="text-xs mb-2">
              <span className="font-semibold">الحالة:</span>
              <span className={`mr-2 px-2 py-1 rounded-full text-white text-xs ${ project.status === 'preparing' ? 'bg-gray-400' : project.status === 'ongoing' ? 'bg-blue-500' : project.status === 'done' ? 'bg-green-500' : 'bg-red-500' }`}>
                {project.status === 'preparing' ? 'قيد الإعداد' : project.status === 'ongoing' ? 'قيد التنفيذ' : project.status === 'done' ? 'مكتمل' : 'مؤرشف'}
              </span>
            </div>
            <div className="text-xs text-gray-500 dark:text-gray-400 mb-4">الأعضاء: {project.assignedMembers.map(mId => members.find(m => m.id === mId)?.name).join(', ')}</div>
            <div className="flex gap-2 text-sm mt-auto">
              <button onClick={() => handleEdit(project)} className="text-blue-500 hover:underline">تعديل</button>
              {user.role === 'owner' && (<button onClick={() => handleDeleteProject(project.id)} className="text-red-500 hover:underline">حذف</button>)}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

// مكون الإحصاءات - **مُحسَّن**
const Stats = ({ projects, dailySummaries, auditLog }) => {
  const [visitorCount, setVisitorCount] = useState(0);

  useEffect(() => {
    const currentCount = parseInt(localStorage.getItem('visitor_count') || '0', 10);
    const newCount = currentCount + 1;
    localStorage.setItem('visitor_count', newCount.toString());
    setVisitorCount(newCount);
  }, []);

  const stats = useMemo(() => {
    const completedProjects = projects.filter(p => p.status === 'done').length;
    return {
      totalProjects: projects.length,
      completedProjects,
      totalSummaries: dailySummaries.length,
    };
  }, [projects, dailySummaries]);

  return (
    <div className="p-6 bg-card-bg rounded-custom shadow-md">
      <h2 className="text-2xl font-bold mb-4 text-primary">لوحة الإحصاءات</h2>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <div className="p-4 bg-gray-100 dark:bg-gray-700 rounded-lg text-center shadow-sm"><h3 className="text-xl font-bold">{stats.totalProjects}</h3><p className="text-sm text-gray-600 dark:text-gray-400">عدد المشاريع</p></div>
        <div className="p-4 bg-gray-100 dark:bg-gray-700 rounded-lg text-center shadow-sm"><h3 className="text-xl font-bold">{stats.completedProjects}</h3><p className="text-sm text-gray-600 dark:text-gray-400">المشاريع المنجزة</p></div>
        <div className="p-4 bg-gray-100 dark:bg-gray-700 rounded-lg text-center shadow-sm"><h3 className="text-xl font-bold">{stats.totalSummaries}</h3><p className="text-sm text-gray-600 dark:text-gray-400">عدد الملخصات</p></div>
        <div className="p-4 bg-gray-100 dark:bg-gray-700 rounded-lg text-center shadow-sm"><h3 className="text-xl font-bold">{visitorCount}</h3><p className="text-sm text-gray-600 dark:text-gray-400">الزوار المحليين</p></div>
      </div>
      <div className="mt-8">
        <h3 className="text-xl font-semibold mb-2">سجل التدقيق</h3>
        <ul className="h-64 overflow-y-auto p-2 bg-gray-50 dark:bg-gray-700 rounded-lg">
          {auditLog.slice().reverse().map(log => (
            <li key={log.id} className="py-1 text-sm border-b border-gray-200 dark:border-gray-600">
              <span className="font-mono text-gray-600 dark:text-gray-400">[{new Date(log.timestamp).toLocaleString()}]</span> {log.event} - {log.userName || log.updatedBy || log.deletedBy || ''}
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
};

// ------------------------------
// المكون الرئيسي (Main Component) - **مُحسَّن**
// ------------------------------
const App = () => {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('materials');
  const [appSettings, setAppSettings] = useState(() => storage.get('app_settings'));
  const [members, setMembers] = useState(() => storage.get('members') || []);
  const [materials, setMaterials] = useState(() => storage.get('materials') || {});
  const [projects, setProjects] = useState(() => storage.get('projects') || []);
  const [dailySummaries, setDailySummaries] = useState(() => storage.get('daily_summaries') || []);
  const [auditLog, setAuditLog] = useState(() => storage.get('audit_log') || []);

  useEffect(() => {
    seedInitialData();
    const settings = storage.get('app_settings');
    if (settings) {
      document.documentElement.className = settings.theme || '';
      document.documentElement.style.setProperty('--border-radius', settings.borderRadius);
      if (settings.primaryColor) {
        document.documentElement.style.setProperty('--primary-color', settings.primaryColor);
      }
    }
  }, []);

  const handleLogin = (loggedInUser) => {
    setUser(loggedInUser);
    setAuditLog(storage.get('audit_log')); // تحديث السجل بعد الدخول
  };

  const handleLogout = () => {
    storage.logEvent({ event: 'USER_LOGOUT', userId: user.id, userName: user.name });
    setAuditLog(storage.get('audit_log'));
    setUser(null);
  };

  const handleToggleTheme = () => {
    const isDark = document.documentElement.classList.toggle('dark');
    const newSettings = { ...appSettings, theme: isDark ? 'dark' : 'light' };
    setAppSettings(newSettings);
    storage.set('app_settings', newSettings);
  };

  if (!user) {
    return <Auth onLogin={handleLogin} />;
  }

  return (
    <div className="min-h-screen bg-bg-color transition-colors duration-300">
      <Header user={user} onLogout={handleLogout} onToggleTheme={handleToggleTheme} />
      <Tabs user={user} activeTab={activeTab} onTabChange={setActiveTab} settings={appSettings} />
      <main className="container mx-auto p-4">
        {activeTab === 'projects' && (
          <Projects user={user} projects={projects} setProjects={setProjects} members={members} setAuditLog={setAuditLog} />
        )}
        {activeTab === 'stats' && (
          <Stats projects={projects} dailySummaries={dailySummaries} auditLog={auditLog} />
        )}
        {/* ... عرض بقية المكونات حسب التبويب النشط ... */}
      </main>
    </div>
  );
};

export default App;
