# 1 Фигуры боевого поля в пятых героях == Вещи в инвентаре персонажа Neverwinter nights

```cs
    public class MatrixModule
    {
        public static bool CanPlaceItem(int[,] ParentMatrix, int[,] subMatrix, int startRow, int startCol)
        {
            int subMatrixRows = subMatrix.GetLength(0);
            int subMatrixCols = subMatrix.GetLength(1);
            int ParentMatrixRows = ParentMatrix.GetLength(0);
            int ParentMatrixCols = ParentMatrix.GetLength(1);

            
            if (startRow + subMatrixRows > ParentMatrixRows || startCol + subMatrixCols > ParentMatrixCols)
            {
                return false;
            }

            for (int i = 0; i < subMatrixRows; i++)
            {
                for (int j = 0; j < subMatrixCols; j++)
                {
                    if (subMatrix[i, j] == 1 && ParentMatrix[startRow + i, startCol + j] == 1)
                    {
                        return false;
                    }
                }
            }

            return true;
        }
    }
```

Данная реализация может относится как к интерфейсу IPlaceable для обьектов на карте поля боя(возможно ли поставить боевую единицу или терраформу), так и для  объекта инвентаря реализующий IMovable.


# Модуль по открытию/закрытию документов определенных расширений
Мы можем реализовать модуль FileManager, который вызывает функцию Open/Close Document (внутри будет контроль лока файла, проверка кеша метаданных файла итд), этот модуль будет удовлетворять интерфейсам IXFormatDocument и IBinaryFormatDocument 
