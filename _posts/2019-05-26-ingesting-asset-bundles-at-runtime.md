---
title: "Ingesting Unity assets at runtime"
author: ArcherN9
date: 2019-05-26 23:00:00 +0100
description: "Consume assets in a unity project at runtime"
category: [Unity]
tags: [Unity, Blob, Augmented Reality, Virtual Reality, Mixed Reality, 3D Models]
---

The web is littered with blog posts, forums and code examples on how to achieve
this. Consume a unity asset - 3D Models, Images, videos at runtime. To download
these from a server and display them to the user, all during runtime. If you've
spent adequate time researching and have eventually arrived at this post,
the following are apparent :

* Majority of the answers are over 2-3 years old; *(as of writing this)*
* Almost all of them use the deprecated `WWW` class;
* They all seem to work for the author and one odd person who responded, but not
you.

## Why would you download assets at runtime?

Unity applications are bulky. Heck, very bulky. I created a simple application
with three 3D models and two scenes using using different Augmented Reality SDKs
which resulted in 50MB+ in APK size. The more complex your application becomes,
the more bloated your app size will be. To mitigate this problem, uploading
assets to a web server and downloading them at runtime to render whatever content
you want is preferable.

## What does not work?

Uploading 3D Models like `.fbx` and `.obj` and downloading them at runtime
attracts an overhead. Unity does not use these assets directly. In lay man terms,
Unity imports these objects, creates meshes & sub meshes, applies textures on
each of the submeshes, etc. All of this is done automatically when you import an
object into Unity. If you were to download these are runtime, all of the above
operations would have to be done manually. That's just something we don't want
to do. Not for our use case.

## Enter, AssetBundles

Asset bundles are containers that contain a map of all the objects you added to
an archive. Once you have an asset Bundle, you can look up Unity synthesized
objects within the archive and use them as you normally would. To create an asset
bundle, follow:

To simplify `AssetBundle` creation, we add an item to the `Assets` menu in the
editor. To do that, we create a new C# script `CreateAssetBundles.cs`.

```cs
public class CreateAssetBundles
{
    // Specifies where the menu item will appear
    [MenuItem("Assets/Build AssetBundles")] 
    static void BuildAllAssetBundles()
    {
        // The folder under which the asset would be generated | Create it if it doesn't exist
        string path = "Assets/AssetBundle";
        if (!Directory.Exists(path))
        {
            Directory.CreateDirectory(path);
        }

        // The AssetBundle that would be generated
        string path2 = "Assets/AssetBundle/myBundle.unity3d";

        // I've used build target as Android. You could change this if required.
        Object[] selection = Selection.GetFiltered(typeof(Object), SelectionMode.DeepAssets);
        _ = BuildPipeline.BuildAssetBundle(Selection.activeObject, selection, path2,
                                       BuildAssetBundleOptions.CollectDependencies
                                     | BuildAssetBundleOptions.CompleteAssets, BuildTarget.Android);
    }
}
```

Once compiled, go to the project structure in the editor and select the 3D
object you would like to bundle in an `AssetBundle`. Go to Assets > Select
`Build AssetBundles`. Give it a few seconds to generate a `.unity3d` file. Go to
your project structure and find the newly created folder `AssetBundle`. Inside,
you should be able to locate a `myBundle.unity3d` file.

![unity Asset Bundle](/assets/img/unity-assetbundles-asset file.png)

I haven't yet figured out how to access this bundle if it is kept a part of the
project. Please consider sending out an email if you have a solution to that.

Now that we have the bundle, we need to host this on a server. You could upload
this anywhere you want. Though I did mine on Azure blob storage. The idea behind
this is to have a truly scalable solution. How to upload to bob storage is beyond
the scope of this post.

Now that you have uploaded the `AssetBundle` and have a URL against it, we can
get down to writing the code for it. The following is a very crude implementation.
You would have to modify this to be able to plug it in an actual application.

```cs
// Use this for initialization
void Start()
{
    StartCoroutine(Load());
}

IEnumerator Load()
{
    
    // Declare the URI where the 3D model is stored 
    string uri = "PATH TO YOUR mybundle.unity3d file";

    
    // WWW is deprecated and a special class that inherits UnityWebRequest is already provided by Unity for
    // specifically our purpose
    UnityWebRequest request = UnityWebRequestAssetBundle.GetAssetBundle(uri);
    
    
    // execute the web request
    yield return request.SendWebRequest();
    
    
    // DownloadHandlerAssetBundle extends DownloadHandler class so we don't have to worry about creating a class, extending it and
    // taking care of the intricacies ourselves
    AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(request);

    
    // If an AssetBundle object is returned, the code successfully loaded the asset. Now, we just have to traverse through the
    // bundle hierarchy and locate our object. 
    // This is because Unity does not export only the 3D model. It exports all of its dependencies with it.
    if (bundle == null)
        Debug.Log("Failed to load AssetBundle!");
    else
    {
        
        // This is not required in the actual program. I did this to figure out all assets stored inside this bundle.
        // The advantage is, you can figure out the hierarchy of the model and locate your prefab.
        foreach (string name in bundle.GetAllAssetNames())
        {
            Debug.Log(name);
        }


        // Load whichever GameObject you want to render on your application
        AndyObject = bundle.LoadAsset<GameObject>("assets/scenes/stage/prefabs/andy.prefab");
    }
}
```
Feature image credits : [cdw.com](blog.cdw.com)
