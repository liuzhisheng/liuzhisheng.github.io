---
layout: post
title: 文章详细页添加最新文章列表，但不包括本文章,hEllo
keywords: WPF
description: 很好啊
excerpt: 很好啊
category: WPF
---

### 高亮Python

{% highlight python linenos %}
@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'
{% endhighlight %}

### 高亮C#
{% highlight C# %}
using System;
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;
using System.ServiceModel;
using System.Text;
using System.Threading;
using System.Windows;
using System.Windows.Media.Imaging;
using Entity;
using Hardcodet.Wpf.TaskbarNotification;
using InternalServiceManager.Interface;
using InternalServiceManager.Service;
using InternalServiceManager.Service.BAL;
using Microsoft.Win32;
using ZQTConsole.AppCode;
using ZQTConsole.View;
using ZQTConsole.ViewModel;
using System.ServiceProcess;

namespace ZQTConsole
{
    /// <summary>
    /// App.xaml 的交互逻辑
    /// </summary>
    public partial class App : Application
    {
        #region 私有变量
        private Mutex mutex;
        private ServiceHost host;
        private Timer informationTimer;
        private Timer newsTimer;
        //private Timer updateUserTimer;
        private Timer csMsgTimer;
        private Timer updateServiceCheckTimer;
        private TaskbarIcon taskbarIcon;
        #endregion

        public App()
        {
            bool flag;
            mutex = new Mutex(true, Constant.MUTEX_NAME_ZQTSERVICE, out flag);
            if (!flag)
            {
                Environment.Exit(-1);
            }
            Directory.SetCurrentDirectory(AppDomain.CurrentDomain.BaseDirectory);
            InitializeComponent();
        }

        #region 目录监视事件

        private void SetupApp(string path)
        {
            try
            {
                ProductMenifest menifest = ProductMenifest.Deserialize(path);
                if (menifest == null)
                {
                    return;
                }

                InstalledApp app = new InstalledApp();
                app.AppID = menifest.ProductID;
                app.Name = menifest.ProductTag;
                app.DisplayName = menifest.ProductName;
                app.Description = menifest.Description;
                app.InstallPath = menifest.UpdateDestDir;
                app.Version = menifest.CurrentVersion;
                app.PublishDate = menifest.DT_PublishDate;
                app.Publisher = menifest.Publisher;
                app.InstallTime = DateTime.Now;

                app.UpdateTime = GetAppLastUpdateTime(app.AppID);

                if (AppManager.ExistsInstallApp(app.AppID))
                {
                    AppManager.UpdateInstallApp(app);
                }
                else
                {
                    AppManager.AddInstallApp(app);
                }
            }
            catch { }
        }

        private void watcher_Created(object sender, FileSystemEventArgs e)
        {
            SetupApp(e.FullPath);
        }

        private void watcher_Changed(object sender, FileSystemEventArgs e)
        {
            SetupApp(e.FullPath);
        }
        #endregion

        #region Base service event
        private void BaseService_CSMessageSync(object sender, CSMessageSyncArgs e)
        {
            if (csMsgTimer == null || e == null)
            {
                return;
            }

            if (e.IsImmediate)
            {
                csMsgTimer.Change(0, 15000);
            }
            else
            {
                csMsgTimer.Change(600000, 1200000);
            }
        }

        private void BaseService_NewUserLogin(object sender, EventArgs e)
        {
            if (!NetParamsMananger.NetParams.ServerEnabled)
            {
                return;
            }

            try
            {
                if (!PacketManager.NetworkTest())
                {
                    return;
                }
            }
            catch
            {
                return;
            }

            List<Entity.User> loginedUsers = BaseService.LoginedUsers;
            foreach (Entity.User u in loginedUsers)
            {
                try
                {
                    PacketManager.SyncInfomation(BaseService.QYH, u.UserID);
                }
                catch (Exception ex)
                {
                    ExceptionManager.LogException(ex);
                }

                //try
                //{
                //    PacketManager.GetRecommendApps(u.UserID);
                //}
                //catch (Exception ex)
                //{
                //    ExceptionManager.LogException(ex);
                //}
            }

            Thread initialTask = new Thread(() =>
            {
                foreach (Entity.User u in loginedUsers)
                {
                    try
                    {
                        PacketManager.DownloadUsers(u.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.DownloadAppRoleUsers(u.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.SyncServerAddress(u.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.DownloadQYZHXX(u.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.SyncKFFeedback(u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.UploadUsers(BaseService.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }

                    try
                    {
                        PacketManager.SyncCustomService(u.QYH, u.UserID);
                    }
                    catch (Exception ex)
                    {
                        ExceptionManager.LogException(ex);
                    }
                }
            });
            initialTask.Priority = ThreadPriority.Highest;
            initialTask.IsBackground = true;
            initialTask.Start();

            informationTimer.Change(600000, 600000);
            csMsgTimer.Change(180000, 1200000);
        }

        #endregion

        private DateTime GetAppLastUpdateTime(string appID)
        {
            DateTime lastUpdateTime = DateTime.Now;
            try
            {
                using (RegistryKey rklm = Registry.LocalMachine)
                {
                    using (RegistryKey rkApp = rklm.OpenSubKey(Constant.REGISTRYKEY_PRODUCT_ROOT + appID))
                    {
                        if (rkApp != null)
                        {
                            lastUpdateTime = DateTime.Parse(rkApp.GetValue(Constant.REGISTRYKEY_PRODUCT_LASTUPDATETIME, string.Empty).ToString());
                        }
                    }
                }
            }
            catch { }

            return lastUpdateTime;
        }

        #region 应用程序事件
        private void Application_Startup(object sender, StartupEventArgs e)
        {
            string zqtPath = string.Empty;
            string addInPath = string.Empty;

            ResourceDictionary rd = new ResourceDictionary
            {
                Source = new Uri("/ZQTConsole;component/Resources/AppRes.xaml", UriKind.Relative)
            };

            taskbarIcon = new TaskbarIcon();

            BitmapImage bitmapim = new BitmapImage(new Uri("pack://application:,,,/Images/logo.ico", UriKind.Absolute));
            taskbarIcon.IconSource = bitmapim;

            taskbarIcon.ContextMenu = (System.Windows.Controls.ContextMenu)rd["contextMenu"];
            taskbarIcon.PopupActivation = PopupActivationMode.All;
            taskbarIcon.MenuActivation = PopupActivationMode.LeftOrRightClick;
            taskbarIcon.Visibility = Visibility.Visible;
            taskbarIcon.ToolTipText = "壳牌快速发票系统";
            taskbarIcon.TrayMouseDoubleClick += new RoutedEventHandler(taskbarIcon_TrayMouseDoubleClick);

            AppViewModel avm = new AppViewModel();
            taskbarIcon.ContextMenu.DataContext = avm;
            BaseService.Restart += avm.OnRestart;
            BaseService.CSMessageSync += new CSMessageSyncHandler(BaseService_CSMessageSync);
            BaseService.NewUserLogin += new EventHandler(BaseService_NewUserLogin);

            #region 监视应用的安装更新
            try
            {
                using (RegistryKey rklm = Registry.LocalMachine)
                {
                    using (RegistryKey rkZqt = rklm.OpenSubKey(Constant.REGISTRYKEY_PRODUCT_ROOT + Constant.PRODUCT_ID_ZQT))
                    {
                        if (rkZqt != null)
                        {
                            zqtPath = rkZqt.GetValue(Constant.REGISTRYKEY_PRODUCT_SETUPPATH, string.Empty).ToString();
                        }
                    }
                }
                addInPath = Path.Combine(zqtPath, Constant.ADDIN_ROOTPATH);
            }
            catch { }

            if (!string.IsNullOrEmpty(addInPath.Trim()) && Directory.Exists(addInPath))
            {
                FileSystemWatcher watcher = new FileSystemWatcher();
                watcher.Path = addInPath;
                watcher.Filter = Constant.APPLICATION_MANIFEST;
                watcher.IncludeSubdirectories = true;
                watcher.Created += new FileSystemEventHandler(watcher_Created);
                watcher.Changed += new FileSystemEventHandler(watcher_Changed);
                watcher.EnableRaisingEvents = true;
            }
            #endregion

            #region 检查是否有新安装或更新的应用
            try
            {
                string[] menifestFiles = Directory.GetFiles(addInPath, Constant.APPLICATION_MANIFEST, SearchOption.AllDirectories);
                foreach (string file in menifestFiles)
                {
                    SetupApp(file);
                }
            }
            catch { }
            #endregion

            #region 启动服务
            try
            {
                host = new ServiceHost(typeof(BaseService));
                host.Open();
            }
            catch
            {
                MessageBox.Show("壳牌快速发票系统启动失败", "错误信息", MessageBoxButton.OK, MessageBoxImage.Error);
                Shutdown();
            }
            #endregion

            #region 启动时的任务
            Thread initialTask = new Thread(() =>
            {
                try
                {
                    PacketManager.NetworkTest();
                }
                catch { }
            });
            initialTask.Priority = ThreadPriority.Highest;
            initialTask.IsBackground = true;
            initialTask.Start();
            #endregion

            #region 同步订阅
            //newsTimer = new Timer((state) =>
            //{
            //    List<Entity.User> loginedUsers = BaseService.LoginedUsers;
            //    try
            //    {
            //        foreach (Entity.User u in loginedUsers)
            //        {
            //            InfomationManager.SyncNews(u.UserID);
            //        }
            //    }
            //    catch (Exception ex)
            //    {
            //        ExceptionManager.LogException(ex);
            //    }
            //}, null, 60000, 300000);
            #endregion

            #region 同步公告、通知
            informationTimer = new Timer((state) =>
            {
                List<Entity.User> loginedUsers = BaseService.LoginedUsers;
                try
                {
                    foreach (Entity.User u in loginedUsers)
                    {
                        PacketManager.SyncInfomation(BaseService.QYH, u.UserID);
                    }
                }
                catch (Exception ex)
                {
                    ExceptionManager.LogException(ex);
                }
            }, null, Timeout.Infinite, 600000);
            #endregion

            //#region 更新用户信息
            //updateUserTimer = new Timer((state) =>
            //{
            //    List<Entity.User> loginedUsers = BaseService.LoginedUsers;
            //    try
            //    {
            //        foreach (Entity.User u in loginedUsers)
            //        {
            //            PacketManager.UploadUsers(BaseService.QYH, u.UserID);
            //        }
            //    }
            //    catch (Exception ex)
            //    {
            //        ExceptionManager.LogException(ex);
            //    }
            //}, null, 60500, 600000);
            //#endregion

            #region 同步客服反馈信息
            csMsgTimer = new Timer((state) =>
            {
                List<Entity.User> loginedUsers = BaseService.LoginedUsers;
                try
                {
                    foreach (Entity.User u in loginedUsers)
                    {
                        PacketManager.SyncKFFeedback(u.UserID);
                    }
                }
                catch (Exception ex)
                {
                    ExceptionManager.LogException(ex);
                }
            }, null, Timeout.Infinite, 1200000);
            #endregion

            #region 检查升级服务
            updateServiceCheckTimer = new Timer((state) =>
            {
                try
                {
                    InitFl.FixService.ChkServiceSt();
                }
                catch (Exception ex)
                {
                    Trace.TraceError("（{0}）：{1}升级服务不能启动：{2}{1}", DateTime.Now, Environment.NewLine, ex.Message);
                }
            }, null, 60000, 180000);
            #endregion
        }

        private void taskbarIcon_TrayMouseDoubleClick(object sender, RoutedEventArgs e)
        {
            Hashtable regApps = BaseService.RegApps;
            if (regApps.Count > 0)
            {
                foreach (DictionaryEntry de in regApps)
                {
                    RegApp ra = (RegApp)de.Value;
                    int pid = ra.ProcessId;

                    Process process = null;
                    try
                    {
                        process = Process.GetProcessById(pid);
                    }
                    catch { }

                    if (process == null)
                    {
                        BaseService.RemoveApp(pid);
                        continue;
                    }
                    SwitchToThisWindow(process.MainWindowHandle, true);
                }
                return;
            }

            string zqtPath = string.Empty;
            try
            {
                using (RegistryKey rklm = Registry.LocalMachine)
                {
                    using (RegistryKey rkZqt = rklm.OpenSubKey(Constant.REGISTRYKEY_PRODUCT_ROOT + Constant.PRODUCT_ID_ZQT))
                    {
                        if (rkZqt != null)
                        {
                            zqtPath = Path.Combine(rkZqt.GetValue(Constant.REGISTRYKEY_PRODUCT_SETUPPATH, string.Empty).ToString(), Constant.EXE_NAME_ZQT);
                        }
                    }
                }
                if (zqtPath.Length > 0)
                {
                    Process.Start(zqtPath);
                }
            }
            catch { }
        }

        private void Application_Exit(object sender, ExitEventArgs e)
        {
            try
            {
                if (host != null)
                {
                    host.Close();
                }

                if (informationTimer != null)
                {
                    informationTimer.Dispose();
                }

                if (newsTimer != null)
                {
                    newsTimer.Dispose();
                }

                //if (updateUserTimer != null)
                //{
                //    updateUserTimer.Dispose();
                //}

                if (csMsgTimer != null)
                {
                    csMsgTimer.Dispose();
                }

                if (updateServiceCheckTimer != null)
                {
                    updateServiceCheckTimer.Dispose();
                }
            }
            catch { }
            taskbarIcon.Dispose();
        }

        private void App_UnhandleException(object sender, System.Windows.Threading.DispatcherUnhandledExceptionEventArgs e)
        {
            StringBuilder error = new StringBuilder();
            ExceptionManager.FormatException(e.Exception, error);

            string errMsg = e.Exception.Message;
            ExceptionForm ef = new ExceptionForm(errMsg, error.ToString());
            ef.ShowDialog();

            ExceptionManager.LogException(e.Exception);
            e.Handled = true;
        }
        #endregion

        #region Win32 API
        [DllImport("User32.dll")]
        private static extern byte SetForegroundWindow(IntPtr hwnd);

        [DllImport("User32.dll")]
        private static extern bool SwitchToThisWindow(IntPtr hwnd, bool fAltTab);
        #endregion
    }
}

{% endhighlight %}