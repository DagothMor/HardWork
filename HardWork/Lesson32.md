# 1 Обработка форм в таблице Excel
## Вызов f# функции в c# приложении со сложными выходными данными
```cs
public bool ExtractFromXLS(Stream FileStream, string newFilePath)
{
    Console.WriteLine("Extracting from xls file...");
    var imageItems = new List<ImageItem>();
    var errors = new List<ExtractError>();
    var result = new ExtractResult(ListModule.OfSeq(imageItems), ListModule.OfSeq(errors),"");

    HSSFWorkbook workbook = new HSSFWorkbook(FileStream);
    var shapesFromSheets = new List<HSSFShape>();


    for (int i = 0; i < workbook.NumberOfSheets; i++)
    {
        ISheet sheet = workbook.GetSheetAt(i);

        HSSFPatriarch patriarch = (HSSFPatriarch)sheet.DrawingPatriarch;

        if (patriarch.CountOfAllChildren == 0) continue;

        shapesFromSheets.AddRange(patriarch.Children);
    }
    // вызов кода на f#
    result = FSharpExtract.extractImages(shapesFromSheets, result);

    for (int count = 0; count < result.ImageItems.Length; count++)
    {
        ImageItem imageItem = result.ImageItems[count];
        try
        {
            string imagePath = Path.Combine(newFilePath, String.IsNullOrEmpty(imageItem.Name)
                ? count.ToString() + "." + imageItem.Image.Format
                : imageItem.Name + "." + imageItem.Image.Format);

            imageItem.Image.Write(imagePath);
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex);
            continue;
        }
    }
    ///...
```

```fsharp
module FSharpExtract

open System
open NPOI.HSSF.UserModel
open ImageMagick

type ImageItem = {
    Name: string
    Image: MagickImage
}

type ExtractError = {
    File: string
    Error: string
}

type ExtractResult = {
    ImageItems: List<ImageItem>
    Errors: List<ExtractError>
    File: String
}


let rec extractImages (children: seq<HSSFShape>) (result: ExtractResult) =
    match Seq.tryHead children with
    | Some (:? HSSFPicture as picture) ->
        try
            let imageFromBytes = new MagickImage(picture.PictureData.Data)
            let imageItem = { Name = picture.FileName; Image = imageFromBytes }
            let newResult = { result with ImageItems = imageItem :: result.ImageItems }
            extractImages (Seq.tail children) newResult
        with | ex ->
            let error = { File = picture.FileName; Error = ex.Message }
            let newResult = { result with Errors = error :: result.Errors }
            extractImages (Seq.tail children) newResult
    | Some _ ->
        extractImages (Seq.tail children) result
    | None -> 
        result
```

Результат

![](Images/Pasted%20image%2020240324220619.png)

Выделив входную структуру данных как список форм во всех таблицах excel, и результат, включающий в себя список массива байтов(картинка) и ошибок(имя картинки, исключение) я смог реализовать на f# метод, который рекурсивно проходится по формам, и в зависимости от формы принимает решение, будем ли мы пытаться преобразовывать в magickImage(напомню не все форматы эта библиотека принимает в linux) или нет(краевый случай как я понимаю).

# 2 Попугай и голодай(Fear and hunger)
```fsharp
type Limb = Head | Torso | Arm | Leg

type LimbHealth = {
    Head: int
    Torso: int
    Arm: int
    Leg: int
}

type Character = {
    Limbs: LimbHealth
    Potions: int
}

type Blow = {
    Damage: int
    TargetLimb: Limb option // None означает, что удар может быть фатальным
}

/// Выпивает зелье для восстановления здоровья всех конечностей.
let usePotion character =
    let healedLimbs = { character.Limbs with
                            Head = character.Limbs.Head + 25
                            Torso = character.Limbs.Torso + 25
                            Arm = character.Limbs.Arm + 25
                            Leg = character.Limbs.Leg + 25 }
    { character with Limbs = healedLimbs; Potions = character.Potions - 1 }

/// Наносит урон конкретной конечности.
let inflictDamage (limbs: LimbHealth) damage targetLimb =
    match targetLimb with
    | Head -> { limbs with Head = limbs.Head - damage }
    | Torso -> { limbs with Torso = limbs.Torso - damage }
    | Arm -> { limbs with Arm = limbs.Arm - damage }
    | Leg -> { limbs with Leg = limbs.Leg - damage }

let isFatalBlow = System.Random().NextDouble() < 0.5

/// Обрабатывает один удар.
let processBlow character (blow: Blow) =
    // Если персонаж уже мертв, дальнейшая обработка не требуется.
    if character.Limbs.Head <= 0 || character.Limbs.Torso <= 0 then character
    else
        if isFatalBlow && blow.TargetLimb.IsNone then
            { character with Limbs = { character.Limbs with Head = 0 } } // Считаем удар фатальным для головы
        else
            match blow.TargetLimb with
            | Some targetLimb ->
                let updatedLimbs = inflictDamage character.Limbs blow.Damage targetLimb
                { character with Limbs = updatedLimbs }
            | None -> character
        |> fun updatedCharacter ->
            if updatedCharacter.Limbs.Head <= 0 || updatedCharacter.Limbs.Torso <= 0 then
                updatedCharacter // Персонаж мёртв, обработка прекращается
            elif updatedCharacter.Limbs.Head < 50 && updatedCharacter.Potions > 0 then
                usePotion updatedCharacter // Использование зелья при критическом уровне здоровья
            else
                updatedCharacter

/// Рекурсивно обрабатывает список ударов.
let rec processBlows character blows =
    match blows with
    | [] -> character.Limbs.Head > 0 && character.Limbs.Torso > 0
    | blow :: restBlows when character.Limbs.Head > 0 && character.Limbs.Torso > 0 ->
        let nextCharacter = processBlow character blow
        processBlows nextCharacter restBlows
    | _ -> false // Если персонаж уже мертв, возвращаем false


let initialCharacter = { Limbs = { Head = 100; Torso = 100; Arm = 100; Leg = 100 }; Potions = 3 }
let blows = [
    { Damage = 10; TargetLimb = Some Head }; // Обычный удар в голову
    { Damage = 15; TargetLimb = None }; // ПокетКэт смотрит вам в душу и показывает самые ужасные страхи.
    { Damage = 20; TargetLimb = Some Torso }; // Удар в торс
    { Damage = 25; TargetLimb = Some Arm } // Удар в руку
]

let isAlive = processBlows initialCharacter blows
printfn "Персонаж жив: %b" isAlive


```

Нужно определить, выживет ли Даан после серии ударов от Покеткэта, выходная структура простая, однако сам процесс ударов по конечностям и возможный фатальный урон делает функцию довольно сложной. 

# Итог
Перед разработкой алгоритма, обязательно нужно согласовывать спецификацию, в частности входные и выходные данные, и по возможности описывать их как рекурсивные, ведь результат в виде списка картинок мы так же можем рекурсивно пройтись и записать в файлы или отталкиваясь от проверки на фатальные серии ударов, нам намного легче преобразовать наш код в уже более серьезный - битву с началом и счастливым/трагическим концом.
