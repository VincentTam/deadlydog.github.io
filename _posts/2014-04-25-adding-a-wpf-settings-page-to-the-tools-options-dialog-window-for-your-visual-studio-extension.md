---
id: 755
title: Adding a WPF Settings Page To The Tools Options Dialog Window For Your Visual Studio Extension
date: 2014-04-25T13:14:44-06:00
guid: http://dans-blog.azurewebsites.net/?p=755
permalink: /adding-a-wpf-settings-page-to-the-tools-options-dialog-window-for-your-visual-studio-extension/
categories:
  - Visual Studio
  - Visual Studio Extensions
tags:
  - Custom
  - Custom Page
  - Options
  - Settings
  - visual studio
  - Visual Studio Extensions
  - Windows Forms
  - WPF
---

I recently created my first Visual Studio extension, [Diff All Files](http://visualstudiogallery.msdn.microsoft.com/d8d61cc9-6660-41af-b8d0-0f8403b4b39c), which allows you to quickly compare the changes to all files in a TFS changeset, shelveset, or pending changes (Git support coming soon). One of the first challenges I faced when I started the project was where to display my extension's settings to the user, and where to save them. My first instinct was to create a new Menu item to launch a page with all of the settings to display, since the wizard you go through to create the project has an option to automatically add a new Menu item the Tools menu. After some Googling though, I found the more acceptable solution is to create a new section within the Tools -> Options window for your extension, as this will also allow the user to [import and export your extension's settings](http://msdn.microsoft.com/en-us/library/bb166176.aspx).

## Adding a grid or custom Windows Forms settings page

Luckily I found this [Stack Overflow answer that shows a Visual Basic example of how to do this](http://stackoverflow.com/a/6247183/602585), and links to [the MSDN page that also shows how to do this in C#](http://msdn.microsoft.com/en-us/library/bb166195.aspx). The MSDN page is a great resource, and it shows you everything you need to create your settings page as either a Grid Page, or a Custom Page using Windows Forms (FYI: when it says to add a UserControl, it means a System.Windows.Forms.UserControl, not a System.Windows.Controls.UserControl). My extension's settings page needed to have buttons on it to perform some operations, which is something the Grid Page doesn't support, so I had to make a Custom Page. I first made it using Windows Forms as the page shows, but it quickly reminded me how out-dated Windows Forms is (no binding!), and my settings page would have to be a fixed width and height, rather than expanding to the size of the users Options dialog window, which I didn't like.

## Adding a custom WPF settings page

The steps to create a Custom WPF settings page are the same as for [creating a Custom Windows Forms Page](http://msdn.microsoft.com/en-us/library/bb166195.aspx), except instead having your settings control inherit from System.Forms.DialogPage (steps 1 and 2 on that page), it needs to inherit from __Microsoft.VisualStudio.Shell.UIElementDialogPage__. And when you create your User Control for the settings page's UI, it will be a WPF System.Windows.Controls.UserControl. Also, instead of overriding the Window method of the DialogPage class, you will override the Child method of the UIElementDialogPage class.

Here's a sample of what the Settings class might look like:

```csharp
using System.Collections.Generic;
using System.ComponentModel;
using System.Linq;
using System.Runtime.InteropServices;
using Microsoft.VisualStudio.Shell;

namespace VS_DiffAllFiles.Settings
{
    [ClassInterface(ClassInterfaceType.AutoDual)]
    [Guid("1D9ECCF3-5D2F-4112-9B25-264596873DC9")]  // Special guid to tell it that this is a custom Options dialog page, not the built-in grid dialog page.
    public class DiffAllFilesSettings : UIElementDialogPage, INotifyPropertyChanged
    {
        #region Notify Property Changed
        /// <summary>
        /// Inherited event from INotifyPropertyChanged.
        /// </summary>
        public event PropertyChangedEventHandler PropertyChanged;

        /// <summary>
        /// Fires the PropertyChanged event of INotifyPropertyChanged with the given property name.
        /// </summary>
        /// <param name="propertyName">The name of the property to fire the event against</param>
        public void NotifyPropertyChanged(string propertyName)
        {
            if (PropertyChanged != null)
                PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
        }
        #endregion

        /// <summary>
        /// Get / Set if new files being added to source control should be compared.
        /// </summary>
        public bool CompareNewFiles { get { return _compareNewFiles; } set { _compareNewFiles = value; NotifyPropertyChanged("CompareNewFiles"); } }
        private bool _compareNewFiles = false;

        #region Overridden Functions

        /// <summary>
        /// Gets the Windows Presentation Foundation (WPF) child element to be hosted inside the Options dialog page.
        /// </summary>
        /// <returns>The WPF child element.</returns>
        protected override System.Windows.UIElement Child
        {
            get { return new DiffAllFilesSettingsPageControl(this); }
        }

        /// <summary>
        /// Should be overridden to reset settings to their default values.
        /// </summary>
        public override void ResetSettings()
        {
            CompareNewFiles = false;
            base.ResetSettings();
        }

        #endregion
    }
}
```

And what the code-behind for the User Control might look like:

```csharp
using System;
using System.Diagnostics;
using System.Linq;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Navigation;

namespace VS_DiffAllFiles.Settings
{
    /// <summary>
    /// Interaction logic for DiffAllFilesSettingsPageControl.xaml
    /// </summary>
    public partial class DiffAllFilesSettingsPageControl : UserControl
    {
        /// <summary>
        /// A handle to the Settings instance that this control is bound to.
        /// </summary>
        private DiffAllFilesSettings _settings = null;

        public DiffAllFilesSettingsPageControl(DiffAllFilesSettings settings)
        {
            InitializeComponent();
            _settings = settings;
            this.DataContext = _settings;
        }

        private void btnRestoreDefaultSettings_Click(object sender, RoutedEventArgs e)
        {
            _settings.ResetSettings();
        }

        private void UserControl_LostKeyboardFocus(object sender, KeyboardFocusChangedEventArgs e)
        {
            // Find all TextBoxes in this control force the Text bindings to fire to make sure all changes have been saved.
            // This is required because if the user changes some text, then clicks on the Options Window's OK button, it closes
            // the window before the TextBox's Text bindings fire, so the new value will not be saved.
            foreach (var textBox in DiffAllFilesHelper.FindVisualChildren<TextBox>(sender as UserControl))
            {
                var bindingExpression = textBox.GetBindingExpression(TextBox.TextProperty);
                if (bindingExpression != null) bindingExpression.UpdateSource();
            }
        }
    }
}
```

And here's the corresponding xaml for the UserControl:

```xml
<UserControl x:Class="VS_DiffAllFiles.Settings.DiffAllFilesSettingsPageControl"
                         xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                         xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                         xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
                         xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
                         xmlns:xctk="http://schemas.xceed.com/wpf/xaml/toolkit"
                         xmlns:QC="clr-namespace:QuickConverter;assembly=QuickConverter"
                         mc:Ignorable="d"
                         d:DesignHeight="350" d:DesignWidth="400" LostKeyboardFocus="UserControl_LostKeyboardFocus">
    <UserControl.Resources>
    </UserControl.Resources>

    <Grid>
        <StackPanel Orientation="Vertical">
            <CheckBox Content="Compare new files" IsChecked="{Binding Path=CompareNewFiles}" ToolTip="If files being added to source control should be compared." />
            <Button Content="Restore Default Settings" Click="btnRestoreDefaultSettings_Click" />
        </StackPanel>
    </Grid>
</UserControl>
```

You can see that I am binding the CheckBox directly to the CompareNewFiles property on the instance of my Settings class; yay, no messing around with Checked events :-).

This is a complete, but very simple example. If you want a more detailed example that shows more controls, check out [the source code for my Diff All Files extension](https://diffallfilesvisualstudioextension.codeplex.com/SourceControl/latest#VS.DiffAllFiles/Settings/DiffAllFilesSettings.cs).

## A minor problem

One problem I found was that when using a TextBox on my Settings Page UserControl, if I edited text in a TextBox and then hit the OK button on the Options dialog to close the window, the new text would not actually get applied. This was because the window would get closed before the TextBox bindings had a chance to fire; so if I instead clicked out of the TextBox before clicking the OK button, everything worked correctly. I know you can change the binding's UpdateSourceTrigger to PropertyChanged, but I perform some additional logic when some of my textbox text is changed, and I didn't want that logic firing after every key press while the user typed in the TextBox.

To solve this problem I added a LostKeyboardFocus event to the UserControl, and in that event I find all TextBox controls on the UserControl and force their bindings to update. You can see the code for this in the snippets above. The one piece of code that's not shown is the FindVisualChildren<TextBox> method, so here it is:

```csharp
/// <summary>
/// Recursively finds the visual children of the given control.
/// </summary>
/// <typeparam name="T">The type of control to look for.</typeparam>
/// <param name="dependencyObject">The dependency object.</param>
public static IEnumerable<T> FindVisualChildren<T>(DependencyObject dependencyObject) where T : DependencyObject
{
    if (dependencyObject != null)
    {
        for (int index = 0; index < VisualTreeHelper.GetChildrenCount(dependencyObject); index++)
        {
            DependencyObject child = VisualTreeHelper.GetChild(dependencyObject, index);
            if (child != null &amp;&amp; child is T)
            {
                yield return (T)child;
            }

            foreach (T childOfChild in FindVisualChildren<T>(child))
            {
                yield return childOfChild;
            }
        }
    }
}
```

And that's it. Now you know how to make a nice Settings Page for your Visual Studio extension using WPF, instead of the archaic Windows Forms.

Happy coding!
